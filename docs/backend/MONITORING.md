# Monitoramento e Observabilidade

Três camadas: **logs estruturados** (Railway), **rastreamento de erros** (Sentry), **uptime** (UptimeRobot).
Sem dependência de serviços pagos além do Sentry free tier e UptimeRobot free tier.

---

## Visão geral

| O que monitorar | Ferramenta | Onde ver |
|-----------------|------------|----------|
| Erros de runtime (exceptions) | Sentry | sentry.io |
| Logs de request / crash | Railway Logs | railway.app → projeto → Logs |
| Uptime do formulário e CRM | UptimeRobot | uptimerobot.com |
| Métricas de negócio (leads/dia, taxa de aprovação) | CRM interno | /crm |
| Performance de banco | Railway Metrics | railway.app → PostgreSQL → Metrics |

---

## Sentry — Rastreamento de erros

### Instalação

```bash
npm install @sentry/nextjs
npx @sentry/wizard@latest -i nextjs
```

O wizard cria `sentry.client.config.ts`, `sentry.server.config.ts` e `sentry.edge.config.ts`.

### Configuração mínima (`sentry.server.config.ts`)

```typescript
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.2,       // 20% das transações
  profilesSampleRate: 0.1,
  beforeSend(event) {
    // nunca enviar dados pessoais do lead para o Sentry
    if (event.request?.data) {
      const d = event.request.data as Record<string, unknown>
      delete d['nome']
      delete d['whatsapp']
      delete d['email']
    }
    return event
  },
})
```

### ENV var necessária

```
SENTRY_DSN=https://xxxxx@o0.ingest.sentry.io/0000000
SENTRY_ORG=sua-org
SENTRY_PROJECT=mentoria-backend
```

### Alertas configurar no Sentry

Regras recomendadas (Settings → Alerts → Create Alert):

| Condição | Ação |
|----------|------|
| Novo issue inédito (first seen) | Notificar imediatamente |
| Error rate > 5% em 5 min | Notificar imediatamente |
| `POST /api/leads` error rate > 10% | Alta prioridade |
| `prisma` no stack trace | Alta prioridade (erro de banco) |

Canais: Sentry envia para **webhook** (sem email). Configure em Settings → Integrations → Webhooks para postar num canal do WhatsApp Business ou Telegram.

---

## Logs estruturados (Railway)

### Como ver logs em produção

Railway Dashboard → Projeto → Serviço `mentoria-backend` → aba **Logs**.

Filtragens úteis:
- `error` — erros de runtime
- `POST /api/leads` — todas as submissões do form
- `429` — IPs que atingiram rate limit
- `prisma` — queries lentas ou falhas de banco

### Logging em código

Next.js escreve para stdout por padrão. Railway captura automaticamente.
Para logs estruturados (mais fáceis de filtrar), use um wrapper simples:

```typescript
// lib/logger.ts
type Level = 'info' | 'warn' | 'error'

export function log(level: Level, event: string, data?: Record<string, unknown>) {
  const entry = {
    ts: new Date().toISOString(),
    level,
    event,
    ...data,
  }
  if (level === 'error') {
    console.error(JSON.stringify(entry))
  } else {
    console.log(JSON.stringify(entry))
  }
}
```

Uso na API:

```typescript
import { log } from '@/lib/logger'

// ao criar lead
log('info', 'lead.created', { score, qualificado: score >= SCORE_THRESHOLD_WA })

// ao rejeitar rate limit
log('warn', 'rate_limit.hit', { ip_prefix: ip.slice(0, 7) }) // nunca IP completo

// em catch global
log('error', 'lead.create.failed', { message: err.message })
```

Resultado no Railway Logs (filtrável por `event`):
```json
{"ts":"2026-06-20T14:32:00.000Z","level":"info","event":"lead.created","score":8,"qualificado":true}
{"ts":"2026-06-20T14:33:00.000Z","level":"warn","event":"rate_limit.hit","ip_prefix":"187.12."}
```

### Logs que NÃO devem aparecer

```
❌ nome, email, whatsapp — nunca em log
❌ DATABASE_URL ou qualquer env var
❌ IP completo (só os primeiros 7 chars)
❌ stack trace com dados de usuário
```

---

## UptimeRobot — Uptime e alertas

Cadastro gratuito em [uptimerobot.com](https://uptimerobot.com).
Monitoramento a cada 5 minutos, alertas via webhook (sem email).

### Monitores a criar

| Nome | URL | Tipo | Intervalo | Alerta |
|------|-----|------|-----------|--------|
| Formulário (público) | `https://aplicar.dominio.com.br/aplicar` | HTTP | 5 min | Webhook |
| CRM (protegido) | `https://aplicar.dominio.com.br/crm` | HTTP | 5 min | Webhook |
| API health | `https://aplicar.dominio.com.br/api/health` | HTTP | 5 min | Webhook |
| Banco PostgreSQL | Railway Health (via API) | Keyword | 10 min | Webhook |

### Endpoint de health check

Criar `app/api/health/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

export async function GET() {
  try {
    await prisma.$queryRaw`SELECT 1`
    return NextResponse.json(
      { status: 'ok', db: 'connected', ts: new Date().toISOString() },
      { status: 200 }
    )
  } catch {
    return NextResponse.json(
      { status: 'error', db: 'disconnected' },
      { status: 503 }
    )
  }
}
```

UptimeRobot monitora `GET /api/health` — se retornar não-200, dispara alerta.

### Webhook de alerta (UptimeRobot → WhatsApp/Telegram)

Configure em UptimeRobot → My Settings → Alert Contacts → Add Alert Contact → Webhook.

**Opção A: Telegram Bot** (mais simples para alertas rápidos)
```
Webhook URL: https://api.telegram.org/bot{BOT_TOKEN}/sendMessage
Método: POST
Body: {"chat_id":"{CHAT_ID}","text":"🚨 *$monitorFriendlyName* está DOWN!\n$alertDetails"}
```

**Opção B: Webhook personalizado** (recebe POST e encaminha para WA Business API)

---

## Métricas de negócio (CRM interno)

O dashboard `/crm` já exibe via `GET /api/dashboard`:

| Métrica | O que observar |
|---------|----------------|
| `totalLeads` | Crescimento diário — se cair, pode ser problema no form ou no tráfego |
| `taxaConversao` | Meta: > 20% de leads qualificados (score ≥ 6) |
| `scoreMedia` | Queda no score médio = tráfego de menor qualidade chegando |
| `leadsPorDia` | Pico/queda abrupta = evento externo ou problema técnico |
| `porStatus.CONVERTIDO` | Principal KPI do negócio |

**Verificar semanalmente:**
- Leads com status `NOVO` há mais de 48h (não analisados) → sinal de backlog
- Leads `QUALIFICADO` sem movimentação → oportunidade perdida

---

## Railway — Métricas de infra

Railway Dashboard → Projeto → Serviço → aba **Metrics**:

| Métrica | Alerta informal |
|---------|----------------|
| CPU > 80% por > 5 min | Investigar query lenta ou loop |
| Memory > 450MB | Possível memory leak |
| Requests/min pico > 200 | Verificar se é tráfego legítimo ou ataque |

Railway Dashboard → PostgreSQL → **Metrics**:

| Métrica | Alerta informal |
|---------|----------------|
| Conexões ativas > 8 | Prisma connection pool saturado |
| Disco > 70% | Planejar expansão ou limpeza de logs |
| Query time P95 > 500ms | Índice faltando ou query N+1 |

---

## Runbook de incidentes

### Formulário retornando 500

1. Railway Logs → filtrar `error` → identificar stack trace
2. Sentry → issue correspondente → ver contexto exato
3. Se for banco: Railway PostgreSQL → Metrics → verificar conexões e disco
4. Rollback rápido: Railway → Deployments → Deploy anterior → "Rollback"

### Taxa de leads zerada (sem submissões por > 2h em horário comercial)

1. UptimeRobot — verificar se o formulário está online
2. Abrir `/aplicar` manualmente e tentar submeter um lead de teste
3. Railway Logs → `POST /api/leads` → últimas ocorrências
4. Verificar se Upstash Redis (rate limit) está operacional em [upstash.com](https://upstash.com)

### CRM inacessível

1. UptimeRobot → monitor do CRM → histórico
2. Railway Logs → erros de auth (`next-auth`)
3. Verificar `NEXTAUTH_SECRET` e `NEXTAUTH_URL` nas env vars do Railway

### Banco PostgreSQL inacessível (`/api/health` retorna 503)

1. Railway → PostgreSQL → Status
2. Se o serviço estiver UP mas a query falhar: verificar `DATABASE_URL` nas env vars
3. Último recurso: Railway → PostgreSQL → Connect → abrir console e rodar `SELECT 1`

---

## Checklist de monitoramento pré-deploy

```
[ ] Sentry DSN configurado nas env vars do Railway
[ ] /api/health retornando 200 com banco conectado
[ ] UptimeRobot monitorando /aplicar, /crm e /api/health
[ ] Webhook de alerta testado (marcar monitor como down manualmente e verificar notificação)
[ ] log('info',...) e log('error',...) aparecendo nos Railway Logs
[ ] Nenhum dado pessoal nos logs (nome, email, whatsapp, IP completo)
[ ] Sentry beforeSend removendo campos pessoais do payload
```
