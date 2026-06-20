# Segurança

Documentação de todas as camadas de segurança do projeto.
Cada camada é independente — uma falha numa não compromete as outras.

---

## Modelo de ameaças

| Ameaça | Vetor | Camada de mitigação |
|--------|-------|---------------------|
| Spam de formulário | Bot/script | Rate limit + Honeypot |
| XSS | Input malicioso | Sanitização + CSP |
| SQL Injection | Input malicioso | Prisma prepared statements |
| CSRF | Cookie theft | SameSite=Strict + CSRF token |
| Clickjacking | iframe | X-Frame-Options DENY + CSP frame-ancestors |
| Força bruta CRM | Login repetido | Lockout Redis após 5 falhas |
| Sessão roubada | Cookie intercept | HTTPS only + Secure flag + HttpOnly |
| Supply chain | CDN comprometido | CSP + SRI onde possível |
| Data breach | Vazamento de banco | Hash de IP, mascaramento de WA no export |
| Acesso não autorizado ao CRM | URL direta | Middleware Next.js + JWT |
| Enumeração de leads | ID previsível | CUID (não incremental) |
| LGPD/multa | Dados pessoais | Endpoint de apagamento + retention policy |

---

## Camada 1 — Formulário (Frontend)

### Honeypot
Campo invisível `name="website"` — escondido via CSS (não `display:none`, que bots detectam):

```css
.honeypot-field {
  opacity: 0;
  position: absolute;
  top: 0;
  left: 0;
  height: 0;
  width: 0;
  z-index: -1;
  pointer-events: none;
}
```

Se preenchido → API retorna `200` falso sem criar o lead no banco.
Bot não sabe que foi rejeitado.

### Double-submit prevention
```typescript
const [isSubmitting, setIsSubmitting] = useState(false)
// botão desabilitado durante requisição
// re-habilita apenas em erro, nunca em sucesso
```

### Sem dados sensíveis em URL
Nunca usar `GET` com dados do formulário. Sempre `POST` com body JSON.

---

## Camada 2 — API (Backend)

### Rate Limiting (`lib/ratelimit.ts`)

```typescript
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

export const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, '15 m'),
  analytics: true,
  prefix: 'mentoria:leads',
})
```

- **5 requests / 15 minutos / IP** para `POST /api/leads`
- **20 requests / 1 minuto / sessão** para endpoints do CRM
- Sliding window (mais preciso que fixed window)

### Validação server-side (Zod)

Nunca confiar no client. O mesmo schema Zod usado no form é revalidado na API:

```typescript
const parsed = leadSchema.safeParse(body)
if (!parsed.success) {
  return NextResponse.json({ error: 'Dados inválidos', fields: ... }, { status: 400 })
}
```

### Sanitização de texto livre (`lib/sanitize.ts`)

```typescript
import { sanitizeHtml } from 'sanitize-html'

export function sanitizeText(input: string): string {
  return sanitizeHtml(input, {
    allowedTags: [],        // sem HTML permitido
    allowedAttributes: {},  // sem atributos
  }).trim()
}
```

Aplicado em `maior_dificuldade`, `por_que_agora`, `notas_internas`.

### SQL Injection
Prisma usa **prepared statements** internamente — nunca concatenação de string em query.
Proibido usar `prisma.$queryRaw` com template literal não parametrizado.

```typescript
// ✅ CORRETO
await prisma.lead.findMany({ where: { email } })

// ❌ NUNCA FAZER
await prisma.$queryRaw`SELECT * FROM "Lead" WHERE email = '${email}'`
```

### Request size limit (`next.config.js`)

```javascript
module.exports = {
  api: {
    bodyParser: {
      sizeLimit: '16kb',  // formulário não precisa de mais
    },
  },
}
```

### HTTP Headers (`next.config.js`)

```javascript
const securityHeaders = [
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline'",  // inline para Next.js
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self'",
      "object-src 'none'",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'",
    ].join('; '),
  },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload',
  },
]

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

---

## Camada 3 — Autenticação do CRM

### NextAuth v5 — Credentials Provider (`lib/auth.ts`)

```typescript
import NextAuth from 'next-auth'
import Credentials from 'next-auth/providers/credentials'
import bcrypt from 'bcrypt'
import { prisma } from './prisma'
import { ratelimit } from './ratelimit'

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      async authorize(credentials) {
        const { email, password } = credentials as { email: string; password: string }

        // Rate limit por email (brute force)
        const { success } = await ratelimit.limit(`login:${email}`)
        if (!success) throw new Error('Muitas tentativas. Aguarde 15 minutos.')

        const admin = await prisma.adminUser.findUnique({ where: { email } })
        if (!admin) throw new Error('Credenciais inválidas')

        // Lockout
        if (admin.lockedUntil && admin.lockedUntil > new Date()) {
          throw new Error('Conta bloqueada temporariamente.')
        }

        const valid = await bcrypt.compare(password, admin.passwordHash)
        if (!valid) {
          await prisma.adminUser.update({
            where: { email },
            data: {
              failedLogins: { increment: 1 },
              lockedUntil: admin.failedLogins >= 4
                ? new Date(Date.now() + 15 * 60 * 1000)
                : null,
            }
          })
          throw new Error('Credenciais inválidas')
        }

        // Reset contadores + registrar login
        await prisma.adminUser.update({
          where: { email },
          data: { failedLogins: 0, lockedUntil: null, lastLoginAt: new Date() }
        })

        return { id: admin.id, email: admin.email, name: admin.name }
      }
    })
  ],
  session: { strategy: 'jwt', maxAge: 8 * 60 * 60 }, // 8h
  pages: { signIn: '/login' },
})
```

### Middleware de proteção (`middleware.ts`)

```typescript
import { auth } from '@/lib/auth'
import { NextResponse } from 'next/server'

export default auth((req) => {
  const isAuthenticated = !!req.auth
  const isCrmRoute = req.nextUrl.pathname.startsWith('/crm')
  const isApiProtected = req.nextUrl.pathname.startsWith('/api/leads') &&
    req.method !== 'POST'  // POST é público (formulário)

  if ((isCrmRoute || isApiProtected) && !isAuthenticated) {
    return NextResponse.redirect(new URL('/login', req.url))
  }
})

export const config = {
  matcher: ['/crm/:path*', '/api/leads/:path*', '/api/dashboard/:path*'],
}
```

---

## Camada 4 — Banco de Dados

### IP nunca em claro

```typescript
import { createHash } from 'crypto'

const ip = req.headers.get('x-forwarded-for')?.split(',')[0] ?? '127.0.0.1'
const ipHash = createHash('sha256').update(ip + process.env.IP_SALT).digest('hex')
// IP_SALT impede rainbow table attack
```

### WhatsApp mascarado no export

```typescript
function maskWhatsapp(wa: string): string {
  // "(88) 99999-4567" → "(88) 9••••-4567"
  return wa.replace(/(\(\d{2}\)\s\d)(\d{4})(-\d{4})/, '$1••••$3')
}
```

### Principle of least privilege (PostgreSQL)

```sql
-- Usuário da aplicação: só SELECT, INSERT, UPDATE, DELETE
-- Sem DROP, ALTER, CREATE, TRUNCATE
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO mentoria_app;
REVOKE CREATE, DROP, ALTER ON SCHEMA public FROM mentoria_app;
```

---

## Camada 5 — LGPD

### Campos de compliance no formulário

```
☐ Li e aceito a Política de Privacidade (link)
```
Campo `consentimento_lgpd: boolean` — obrigatório, salvo no banco com timestamp.

### Endpoint de apagamento

`DELETE /api/leads/[id]/gdpr` — anonimiza dados pessoais sem deletar o registro
(mantém métricas de score/status para analytics).

### Retention policy (cron)

Leads PERDIDO ou NAO_QUALIFICADO → apagados após 90 dias.
Configurar no Railway como cron job ou GitHub Actions Schedule.

### O que NÃO armazenar

- IP em claro (só hash SHA-256 + salt)
- Senha em qualquer forma legível (só bcrypt hash)
- Dados bancários (nunca coletamos)
- Documentos pessoais (nunca coletamos)

---

## Checklist de segurança pré-deploy

```
[ ] Todas as env vars no Railway, nunca no código
[ ] DATABASE_URL não exposta em logs
[ ] NEXTAUTH_SECRET com >= 32 chars aleatórios
[ ] IP_SALT com >= 16 chars aleatórios
[ ] npm audit --audit-level=high sem vulnerabilidades
[ ] Headers HTTP validados via securityheaders.com
[ ] Rate limit testado (5 req seguidas devem retornar 429)
[ ] Honeypot testado (form com campo preenchido → 200 sem criar lead)
[ ] Login com senha errada 6x → lockout funcionando
[ ] Rota /crm sem login → redirect para /login
[ ] DELETE GDPR testado → campos anonimizados
[ ] HTTPS funcionando (Railway TLS automático)
[ ] robots.txt bloqueando /api/*
```
