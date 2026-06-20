# Variáveis de Ambiente

---

## `.env.example` (commitar no repositório)

```bash
# ─────────────────────────────────────────────────────────
# BANCO DE DADOS (injetado automaticamente pelo Railway)
# ─────────────────────────────────────────────────────────
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/mentoria"

# ─────────────────────────────────────────────────────────
# NEXTAUTH
# ─────────────────────────────────────────────────────────
NEXTAUTH_URL="https://seudominio.com.br"
# Gerar: openssl rand -base64 32
NEXTAUTH_SECRET=""

# ─────────────────────────────────────────────────────────
# UPSTASH REDIS (rate limiting)
# ─────────────────────────────────────────────────────────
UPSTASH_REDIS_REST_URL=""
UPSTASH_REDIS_REST_TOKEN=""

# ─────────────────────────────────────────────────────────
# SEGURANÇA
# ─────────────────────────────────────────────────────────
# Gerar: openssl rand -base64 16
IP_SALT=""

# ─────────────────────────────────────────────────────────
# EMAIL (Resend)
# ─────────────────────────────────────────────────────────
RESEND_API_KEY=""
NOTIFY_EMAIL="cadu@seudominio.com.br"

# ─────────────────────────────────────────────────────────
# ADMIN INICIAL (usado apenas no seed — remover depois)
# ─────────────────────────────────────────────────────────
ADMIN_EMAIL="cadu@seudominio.com.br"
ADMIN_INITIAL_PASSWORD=""

# ─────────────────────────────────────────────────────────
# PÚBLICAS (prefixo NEXT_PUBLIC_ → visíveis no browser)
# ─────────────────────────────────────────────────────────
NEXT_PUBLIC_WA_NUMBER="5588999471567"
NEXT_PUBLIC_APP_URL="https://seudominio.com.br"
```

---

## Regras

| Regra | Motivo |
|-------|--------|
| Nunca commitar `.env` ou `.env.local` | Exporia secrets no Git |
| `.env.example` sem valores reais | Documentação segura de quais vars existem |
| `NEXT_PUBLIC_*` apenas para dados não-sensíveis | São expostas no bundle do browser |
| Cada ambiente tem seu próprio set de vars | Produção e staging nunca compartilham secrets |
| Rotacionar `NEXTAUTH_SECRET` a cada 6 meses | Boa prática de segurança |

---

## Como gerar valores seguros

```bash
# NEXTAUTH_SECRET (32+ bytes)
openssl rand -base64 32

# IP_SALT (16+ bytes)
openssl rand -base64 16

# Senha do admin inicial (25+ chars)
openssl rand -base64 20
```

---

## Onde cada var é usada

| Variável | Arquivo | Uso |
|----------|---------|-----|
| `DATABASE_URL` | `lib/prisma.ts` | Conexão Prisma |
| `NEXTAUTH_URL` | NextAuth internamente | URL base para callbacks |
| `NEXTAUTH_SECRET` | NextAuth internamente | Assina JWT |
| `UPSTASH_REDIS_REST_URL` | `lib/ratelimit.ts` | Conexão Redis |
| `UPSTASH_REDIS_REST_TOKEN` | `lib/ratelimit.ts` | Auth Redis |
| `IP_SALT` | `app/api/leads/route.ts` | Hash do IP |
| `RESEND_API_KEY` | `lib/notify.ts` | Envio de email |
| `NOTIFY_EMAIL` | `lib/notify.ts` | Destinatário da notificação |
| `ADMIN_EMAIL` | `prisma/seed.ts` | Criar admin inicial |
| `ADMIN_INITIAL_PASSWORD` | `prisma/seed.ts` | Senha do admin inicial |
| `NEXT_PUBLIC_WA_NUMBER` | Componentes React | Link do WhatsApp |
| `NEXT_PUBLIC_APP_URL` | Componentes React | URL base da aplicação |
