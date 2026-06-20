# Deploy — Railway

Guia completo do zero ao ar em produção.

---

## Pré-requisitos

- Conta no [Railway](https://railway.app) (plano **Hobby $5/mês** — obrigatório para produção)
- Conta no [GitHub](https://github.com) com o repositório do projeto
- Conta no [Upstash](https://upstash.com) (free tier — Redis para rate limiting)
- Conta no [Resend](https://resend.com) (free tier — 3k emails/mês)
- Domínio próprio (ex: `dominio.com.br`) com acesso ao DNS

---

## Passo 1 — Preparar o repositório

```bash
# Criar projeto
npx create-next-app@latest mentoria-backend \
  --typescript --tailwind --app --no-src-dir --import-alias "@/*"

cd mentoria-backend

# Instalar dependências de produção
npm install \
  @prisma/client \
  next-auth@beta \
  bcrypt \
  zod \
  react-hook-form \
  @hookform/resolvers \
  framer-motion \
  @upstash/redis \
  @upstash/ratelimit \
  resend \
  sanitize-html \
  recharts

npm install -D \
  prisma \
  @types/bcrypt \
  @types/sanitize-html

# Inicializar Prisma
npx prisma init --datasource-provider postgresql

# Instalar shadcn/ui
npx shadcn@latest init --defaults

# Adicionar componentes necessários
npx shadcn@latest add \
  button input label select textarea badge card \
  table tabs dialog sheet accordion progress \
  separator skeleton tooltip avatar

# Criar .gitignore adequado
cat > .gitignore << 'EOF'
node_modules/
.next/
.env
.env.local
.env.production
*.log
.DS_Store
EOF

git init
git add .
git commit -m "chore: initial setup"
git remote add origin https://github.com/cadubezerra/mentoria-backend
git push -u origin main
```

---

## Passo 2 — Railway: criar projeto e banco

```
1. Acesse railway.app → New Project
2. "Deploy from GitHub repo" → selecione mentoria-backend
3. Railway detecta Next.js automaticamente
4. Clique em "+ Add Service" → Database → PostgreSQL
   → Railway cria o banco e injeta DATABASE_URL automaticamente
5. No serviço Next.js → Variables → veja que DATABASE_URL já está lá
```

---

## Passo 3 — Variáveis de ambiente no Railway

No serviço Next.js → **Variables** → adicione uma por uma:

```bash
# NextAuth
NEXTAUTH_URL=https://seudominio.com.br
NEXTAUTH_SECRET=<gere com: openssl rand -base64 32>

# Upstash Redis
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=AXxx...

# IP Salt (para hash de IP)
IP_SALT=<gere com: openssl rand -base64 16>

# Resend (email)
RESEND_API_KEY=re_xxx...
NOTIFY_EMAIL=cadu@seudominio.com.br

# Admin seed (usado apenas uma vez)
ADMIN_EMAIL=cadu@seudominio.com.br
ADMIN_INITIAL_PASSWORD=<senha forte — mude após primeiro login>

# App
NEXT_PUBLIC_WA_NUMBER=5588999471567
NEXT_PUBLIC_APP_URL=https://seudominio.com.br
```

> **Nunca** adicione `DATABASE_URL` manualmente — Railway injeta automaticamente.

---

## Passo 4 — Migrations e seed

```bash
# Instalar Railway CLI
npm install -g @railway/cli

# Login
railway login

# Linkar ao projeto
railway link

# Rodar migrations em produção
railway run npx prisma migrate deploy

# Criar admin inicial (apenas uma vez)
railway run npx prisma db seed
```

---

## Passo 5 — Domínio customizado

```
1. Railway → serviço Next.js → Settings → Domains
2. "Add Custom Domain" → seudominio.com.br
3. Railway mostra o CNAME/A record necessário
4. Adicionar no painel DNS do seu registrador (Registro.br / GoDaddy / Cloudflare)
5. Aguardar propagação (até 24h, geralmente < 1h)
6. Railway provisiona TLS via Let's Encrypt automaticamente
```

### Configuração recomendada de subdomínios

| Subdomínio | Destino |
|-----------|---------|
| `seudominio.com.br` | LP estática (index.html) — hospedagem atual |
| `aplicar.seudominio.com.br` | Formulário (Railway Next.js) |
| `crm.seudominio.com.br` | Painel CRM (Railway Next.js — mesma app) |

O mesmo serviço Railway serve os dois subdomínios — adicione os dois em Domains.
Next.js middleware diferencia `/aplicar/*` de `/crm/*`.

---

## Passo 6 — CI/CD automático

Railway faz deploy automático a cada push na branch `main`.

```bash
# Fluxo de trabalho recomendado:
git checkout -b feature/minha-feature
# ... desenvolve ...
git push origin feature/minha-feature
# Railway cria preview deploy automático para a branch
# Testa no preview URL
# Abre PR → merge na main
# Railway faz deploy em produção automaticamente
```

---

## Variáveis por ambiente

Railway suporta múltiplos environments (Production / Staging).

```
Production:  main branch  →  domínio real
Staging:     develop branch → subdomínio de staging
```

---

## Monitoramento

```
Railway → serviço → Metrics:
  - CPU usage
  - Memory usage
  - Requests/min
  - Error rate

Railway → serviço → Logs:
  - Logs em tempo real
  - Filtro por nível (error, warn, info)
```

### Alertas recomendados

Configurar via Railway Webhooks ou integração com Slack/email:
- Error rate > 5% em 5 minutos
- Memory > 80% por 10 minutos
- Deploy falhou

---

## Backup do banco

```bash
# Manual (a qualquer momento)
railway run pg_dump $DATABASE_URL > backup-$(date +%Y%m%d).sql

# Railway Hobby: backup automático diário, retenção 7 dias
# Railway Pro: retenção 30 dias
```

---

## Rollback

```bash
# Ver deploys anteriores
railway deployments

# Rollback para deploy específico
railway rollback <deployment-id>
```

---

## Comandos úteis do dia a dia

```bash
# Ver logs em tempo real
railway logs --tail

# Acessar banco diretamente
railway run psql $DATABASE_URL

# Rodar script pontual
railway run npx ts-node scripts/cleanup-old-leads.ts

# Ver todas as env vars (mascaradas)
railway variables

# Abrir Prisma Studio no banco de produção
railway run npx prisma studio
```

---

## Checklist pré-go-live

```
[ ] DATABASE_URL injetada pelo Railway (não manual)
[ ] Migrations rodadas: railway run npx prisma migrate deploy
[ ] Seed rodado: railway run npx prisma db seed
[ ] Login no CRM funcionando com ADMIN_EMAIL
[ ] ADMIN_INITIAL_PASSWORD trocada após primeiro acesso
[ ] Formulário em aplicar.seudominio.com.br acessível
[ ] CRM em crm.seudominio.com.br acessível (redireciona para /login sem auth)
[ ] Rate limit testado: 6 submissões seguidas → 429
[ ] Email de notificação chegando no NOTIFY_EMAIL
[ ] HTTPS ativo nos dois subdomínios
[ ] Headers de segurança: securityheaders.com nota A ou A+
[ ] robots.txt bloqueando /api/*
[ ] npm audit limpo antes do merge na main
```
