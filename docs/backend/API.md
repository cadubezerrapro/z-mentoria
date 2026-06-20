# API — Contratos

Base URL produção: `https://api.dominio.com.br`
Todos os endpoints retornam `Content-Type: application/json`.
Endpoints marcados com `🔒` exigem sessão NextAuth válida (cookie `next-auth.session-token`).

---

## `POST /api/leads`

Recebe a submissão do formulário. Público (sem auth). Protegido por rate limit.

### Rate limit
- **5 requisições / 15 minutos / IP**
- Identificado pelo header `x-forwarded-for` (Railway usa proxy)
- Implementado com `@upstash/ratelimit` + Sliding Window

### Request

```http
POST /api/leads
Content-Type: application/json

{
  "nome": "João Silva",
  "whatsapp": "(88) 99999-4567",
  "email": "joao@email.com",
  "comoEncontrou": "Instagram",

  "oque_vende": "Mentoria de emagrecimento",
  "segmento": "Saúde",
  "tempo_mercado": "1–3 anos",
  "vende_internacional": "BR + Internacional",

  "faturamento_atual": "R$15K–R$50K",
  "budget_trafego": "R$3K–R$10K",
  "tem_reserva": "Sim, tenho capital disponível",
  "fonte_vendas": "Misto",

  "rodou_trafego": "Sim, com frequência",
  "nivel_meta_ads": 4,
  "ja_faturou_10k": true,
  "usa_funil": "Sim, completo",
  "ferramentas": ["Hotmart", "Manychat"],

  "tem_computador": "Sim, rápido",
  "horas_semana": "10–20h",
  "tem_cnpj": "Sim",
  "tem_equipe": "Não, só eu",

  "maior_dificuldade": "Não consigo escalar o tráfego sem perder margem",
  "por_que_agora": "Quero faturar em dólar antes do fim do ano",
  "meta_90dias": "R$50K",
  "comprometimento": "100% comprometido",

  "consentimento_lgpd": true,
  "honeypot": "",

  "utm_source": "instagram",
  "utm_medium": "cpc",
  "utm_campaign": "underground-br"
}
```

### Responses

**201 Created — lead qualificado (score >= 6)**
```json
{
  "success": true,
  "score": 9,
  "redirectToWhatsApp": true
}
```

**201 Created — lead não qualificado (score < 6)**
```json
{
  "success": true,
  "score": 4,
  "redirectToWhatsApp": false
}
```

**400 Bad Request — validação falhou**
```json
{
  "success": false,
  "error": "Dados inválidos",
  "fields": {
    "email": "E-mail inválido",
    "whatsapp": "Formato: (88) 99999-9999"
  }
}
```

**429 Too Many Requests — rate limit**
```json
{
  "success": false,
  "error": "Muitas tentativas. Tente novamente em 15 minutos.",
  "retryAfter": 900
}
```

**200 OK — honeypot preenchido (resposta falsa, não revela detecção)**
```json
{
  "success": true,
  "score": 5,
  "redirectToWhatsApp": false
}
```

### Implementação (`app/api/leads/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { ratelimit } from '@/lib/ratelimit'
import { leadSchema } from '@/lib/validations/lead.schema'
import { prisma } from '@/lib/prisma'
import { calcularScore, SCORE_THRESHOLD_WA } from '@/lib/score'
import { sendLeadNotification } from '@/lib/notify'
import { createHash } from 'crypto'

export async function POST(req: NextRequest) {
  // 1. Rate limit por IP
  const ip = req.headers.get('x-forwarded-for')?.split(',')[0] ?? '127.0.0.1'
  const { success, reset } = await ratelimit.limit(ip)

  if (!success) {
    return NextResponse.json(
      { success: false, error: 'Muitas tentativas.', retryAfter: Math.floor((reset - Date.now()) / 1000) },
      { status: 429 }
    )
  }

  // 2. Parse + validação Zod
  const body = await req.json().catch(() => null)
  if (!body) return NextResponse.json({ success: false, error: 'Body inválido' }, { status: 400 })

  const parsed = leadSchema.safeParse(body)
  if (!parsed.success) {
    const fields = Object.fromEntries(
      parsed.error.issues.map(i => [i.path.join('.'), i.message])
    )
    return NextResponse.json({ success: false, error: 'Dados inválidos', fields }, { status: 400 })
  }

  const data = parsed.data

  // 3. Honeypot — resposta falsa, sem processar
  if (data.honeypot && data.honeypot.length > 0) {
    return NextResponse.json({ success: true, score: 3, redirectToWhatsApp: false })
  }

  // 4. Score
  const { total: score, breakdown } = calcularScore(data)

  // 5. Salvar no banco
  const ipHash = createHash('sha256').update(ip).digest('hex')
  const uaHash = createHash('sha256')
    .update(req.headers.get('user-agent') ?? '')
    .digest('hex')

  await prisma.lead.create({
    data: {
      ...data,
      score,
      score_breakdown: breakdown,
      ip_hash: ipHash,
      user_agent_hash: uaHash,
      ferramentas: data.ferramentas,
      consentimento_lgpd: data.consentimento_lgpd,
    }
  })

  // 6. Notificar Cadu via email
  await sendLeadNotification({ nome: data.nome, score, whatsapp: data.whatsapp })

  return NextResponse.json(
    { success: true, score, redirectToWhatsApp: score >= SCORE_THRESHOLD_WA },
    { status: 201 }
  )
}
```

---

## `GET /api/leads` 🔒

Lista de leads com filtros e paginação.

### Query params

| Param | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `page` | number | `1` | Página atual |
| `limit` | number | `20` | Itens por página (max 100) |
| `status` | LeadStatus | todos | Filtro por status |
| `score_min` | number 0–10 | `0` | Score mínimo |
| `search` | string | — | Busca em nome, email, whatsapp |
| `segmento` | string | — | Filtro por segmento |
| `order` | `asc`\|`desc` | `desc` | Ordenação por createdAt |

### Response `200`

```json
{
  "leads": [
    {
      "id": "clx...",
      "nome": "João Silva",
      "whatsapp": "(88) 9••••-4567",
      "email": "joao@email.com",
      "score": 9,
      "status": "QUALIFICADO",
      "faturamento_atual": "R$15K–R$50K",
      "segmento": "Saúde",
      "createdAt": "2026-06-20T14:32:00.000Z"
    }
  ],
  "total": 247,
  "page": 1,
  "pages": 13,
  "limit": 20
}
```

---

## `GET /api/leads/[id]` 🔒

Retorna lead completo com histórico de status.

### Response `200`

```json
{
  "lead": {
    "id": "clx...",
    "nome": "João Silva",
    "score": 9,
    "score_breakdown": {
      "financeiro": 3.5,
      "experiencia": 2.8,
      "operacional": 1.75,
      "comprometimento": 1
    },
    "status": "QUALIFICADO",
    "notas_internas": "Chamada agendada para amanhã",
    "statusHistory": [
      {
        "status": "QUALIFICADO",
        "changedAt": "2026-06-20T14:45:00.000Z",
        "nota": null
      },
      {
        "status": "NOVO",
        "changedAt": "2026-06-20T14:32:00.000Z",
        "nota": null
      }
    ]
    // ... todos os campos do lead
  }
}
```

### Response `404`

```json
{ "error": "Lead não encontrado" }
```

---

## `PATCH /api/leads/[id]` 🔒

Atualiza status e/ou notas.

### Request

```json
{
  "status": "EM_NEGOCIACAO",
  "nota": "Chamada agendada para amanhã às 15h",
  "notas_internas": "Tem budget, só precisa de aprovação do sócio"
}
```

Todos os campos são opcionais — só envia o que mudar.

### Response `200`

```json
{
  "success": true,
  "lead": { "id": "clx...", "status": "EM_NEGOCIACAO", "updatedAt": "..." }
}
```

---

## `DELETE /api/leads/[id]/gdpr` 🔒

Remove todos os dados pessoais do lead (direito ao esquecimento — LGPD Art. 18).
Não deleta o registro, anonimiza os campos pessoais.

### Response `200`

```json
{
  "success": true,
  "message": "Dados pessoais removidos conforme LGPD"
}
```

O lead fica no banco como:
```
nome: "[removido]"
whatsapp: "[removido]"
email: "[removido]"
apagado_em: "2026-06-20T15:00:00.000Z"
```

---

## `GET /api/dashboard` 🔒

Métricas para o dashboard.

### Response `200`

```json
{
  "totalLeads": 247,
  "hoje": 12,
  "estaSemana": 43,
  "qualificados": 89,
  "convertidos": 23,
  "taxaConversao": "25.8%",
  "scoreMedia": 6.4,
  "leadsPorDia": [
    { "date": "2026-06-14", "count": 8 },
    { "date": "2026-06-15", "count": 11 },
    { "date": "2026-06-16", "count": 6 },
    { "date": "2026-06-17", "count": 14 },
    { "date": "2026-06-18", "count": 9 },
    { "date": "2026-06-19", "count": 7 },
    { "date": "2026-06-20", "count": 12 }
  ],
  "distribuicaoScore": [
    { "score": 1, "count": 4 },
    { "score": 5, "count": 32 },
    { "score": 9, "count": 18 }
  ],
  "porStatus": {
    "NOVO": 45,
    "EM_ANALISE": 28,
    "QUALIFICADO": 89,
    "NAO_QUALIFICADO": 62,
    "EM_NEGOCIACAO": 12,
    "CONVERTIDO": 23,
    "PERDIDO": 11
  }
}
```

---

## `GET /api/leads/export` 🔒

Baixa todos os leads (com filtros opcionais) como CSV.

```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="leads-2026-06-20.csv"
```

Colunas: `id, nome, whatsapp, email, score, status, faturamento_atual, segmento, createdAt`

WA exportado como `(88) 9••••-4567` por padrão.
Para exportar completo, exige confirmação (`?unmask=true`).
