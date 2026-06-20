# Formulário de Qualificação

Formulário multi-step de 6 etapas. Cada etapa é um componente React isolado.
O estado global fica em `MultiStepForm.tsx` via `useReducer`.
Validação client-side com Zod + React Hook Form; mesmos schemas revalidados no servidor.

---

## UX e fluxo

```
[Topo]  Barra de progresso  ████████░░░░  Etapa 3 de 6

[Corpo] Pergunta + campo(s)

[Rodapé]
  ← Voltar          Próximo →
  (disabled no step 1)    (submit no step 6)
```

Transição entre etapas: `framer-motion` com `AnimatePresence` (slide horizontal).
Sem recarregamento de página. Dados persistem no estado local durante navegação entre etapas.

---

## Schema Zod completo (`lib/validations/lead.schema.ts`)

```typescript
import { z } from 'zod'

const step1Schema = z.object({
  nome: z.string().min(2, 'Mínimo 2 caracteres').max(100),
  whatsapp: z
    .string()
    .regex(/^\(\d{2}\)\s9\d{4}-\d{4}$/, 'Formato: (88) 99999-9999'),
  email: z.string().email('E-mail inválido'),
  comoEncontrou: z.enum(['Instagram', 'Indicação', 'Google', 'Outro']).optional(),
})

const step2Schema = z.object({
  oque_vende: z.string().min(3).max(200),
  segmento: z.enum(['Saúde', 'Finanças', 'Relacionamento', 'Negócios', 'Educação', 'Outro']),
  tempo_mercado: z.enum(['< 6 meses', '6m–1ano', '1–3 anos', '3+ anos']),
  vende_internacional: z.enum(['Só BR', 'BR + Internacional', 'Só Internacional']),
})

const step3Schema = z.object({
  faturamento_atual: z.enum(['< R$5K', 'R$5K–R$15K', 'R$15K–R$50K', 'R$50K+', 'Não tenho ainda']),
  budget_trafego: z.enum(['< R$1K', 'R$1K–R$3K', 'R$3K–R$10K', 'R$10K+']),
  tem_reserva: z.enum(['Sim, tenho capital disponível', 'Ainda estou juntando', 'Não tenho']),
  fonte_vendas: z.enum(['Orgânico', 'Tráfego pago', 'Indicação', 'Evento/Lançamento', 'Misto']),
})

const step4Schema = z.object({
  rodou_trafego: z.enum(['Sim, com frequência', 'Já tentei / parei', 'Nunca rodei']),
  nivel_meta_ads: z.number().int().min(1).max(5),
  ja_faturou_10k: z.boolean(),
  usa_funil: z.enum(['Sim, completo', 'Parcial', 'Não uso']),
  ferramentas: z.array(
    z.enum(['ActiveCampaign', 'Manychat', 'Hotmart', 'Kiwify', 'Nenhuma', 'Outra'])
  ).min(1, 'Selecione ao menos uma opção'),
})

const step5Schema = z.object({
  tem_computador: z.enum(['Sim, rápido', 'Sim, mas lento', 'Só celular']),
  horas_semana: z.enum(['< 5h', '5–10h', '10–20h', '20h+']),
  tem_cnpj: z.enum(['Sim', 'Não, só PF', 'Em processo']),
  tem_equipe: z.enum(['Sim', 'Não, só eu', 'Terceirizado']),
})

const step6Schema = z.object({
  maior_dificuldade: z.string().min(10, 'Descreva com mais detalhes').max(300),
  por_que_agora: z.string().min(10).max(300),
  meta_90dias: z.enum(['R$10K', 'R$30K', 'R$50K', 'R$100K', 'R$100K+']),
  comprometimento: z.enum(['100% comprometido', 'Ainda avaliando', 'Depende do valor']),
  consentimento_lgpd: z.literal(true, {
    errorMap: () => ({ message: 'Você deve aceitar a Política de Privacidade' })
  }),
  honeypot: z.string().max(0, '').optional(), // deve ser vazio
})

// Schema completo para validação server-side
export const leadSchema = step1Schema
  .merge(step2Schema)
  .merge(step3Schema)
  .merge(step4Schema)
  .merge(step5Schema)
  .merge(step6Schema)

export type LeadFormData = z.infer<typeof leadSchema>

// Schemas por etapa (para validação progressiva no client)
export const stepSchemas = [
  step1Schema,
  step2Schema,
  step3Schema,
  step4Schema,
  step5Schema,
  step6Schema,
] as const
```

---

## Campos por etapa

### Etapa 1 — Identificação

| Campo | Tipo | Máscara / Validação |
|-------|------|---------------------|
| Nome completo | `Input` | min 2 chars |
| WhatsApp | `Input` com máscara | `(99) 99999-9999` |
| E-mail | `Input type="email"` | formato válido |
| Como encontrou Cadu | `Select` opcional | 4 opções |

### Etapa 2 — Sobre o negócio

| Campo | Tipo | Opções |
|-------|------|--------|
| O que você vende | `Input` | texto livre, max 200 |
| Segmento | `Select` | Saúde · Finanças · Relacionamento · Negócios · Educação · Outro |
| Há quanto tempo atua | `RadioGroup` | < 6m · 6m–1a · 1–3a · 3a+ |
| Mercado de atuação | `RadioGroup` | Só BR · BR + Internacional · Só Internacional |

### Etapa 3 — Financeiro

| Campo | Tipo | Opções |
|-------|------|--------|
| Faturamento atual/mês | `RadioGroup` | < R$5K · R$5K–R$15K · R$15K–R$50K · R$50K+ · Não tenho |
| Budget mensal p/ tráfego | `RadioGroup` | < R$1K · R$1K–R$3K · R$3K–R$10K · R$10K+ |
| Tem reserva p/ investir | `RadioGroup` | Sim · Juntando · Não tenho |
| Principal fonte de vendas | `Select` | Orgânico · Tráfego pago · Indicação · Lançamento · Misto |

### Etapa 4 — Experiência com tráfego

| Campo | Tipo | Detalhe |
|-------|------|---------|
| Já rodou tráfego pago? | `RadioGroup` | Sim frequente · Tentei/parei · Nunca |
| Nível Meta Ads | `Slider` 1–5 | Labels: Zero / Básico / Intermediário / Avançado / Expert |
| Já faturou > R$10K/mês? | `Switch` | Sim / Não |
| Usa funil de vendas? | `RadioGroup` | Sim completo · Parcial · Não uso |
| Ferramentas que usa | `CheckboxGroup` | ActiveCampaign · Manychat · Hotmart · Kiwify · Nenhuma · Outra |

### Etapa 5 — Operacional

| Campo | Tipo | Opções |
|-------|------|--------|
| Tem PC ou notebook? | `RadioGroup` | Sim rápido · Sim lento · Só celular |
| Horas disponíveis/semana | `RadioGroup` | < 5h · 5–10h · 10–20h · 20h+ |
| Tem CNPJ / conta PJ? | `RadioGroup` | Sim · Não/PF · Em processo |
| Tem assistente ou equipe? | `RadioGroup` | Sim · Só eu · Terceirizado |

### Etapa 6 — Motivação + Consentimento

| Campo | Tipo | Detalhe |
|-------|------|---------|
| Maior dificuldade hoje | `Textarea` | min 10 · max 300 chars, counter visível |
| Por que quer entrar agora | `Textarea` | min 10 · max 300 chars |
| Meta de faturamento 90 dias | `RadioGroup` | R$10K · R$30K · R$50K · R$100K · R$100K+ |
| Comprometimento | `RadioGroup` | 100% comprometido · Avaliando · Depende do valor |
| Aceite LGPD | `Checkbox` | obrigatório, com link para /privacidade |
| honeypot | `Input` escondido via CSS | deve permanecer vazio |

---

## Algoritmo de score (`lib/score.ts`)

```typescript
export interface ScoreBreakdown {
  financeiro: number     // max 4
  experiencia: number    // max 3
  operacional: number    // max 2
  comprometimento: number // max 1
}

export function calcularScore(data: LeadFormData): {
  total: number
  breakdown: ScoreBreakdown
} {
  // ─── FINANCEIRO (0–4) ──────────────────────────────────────
  const fatMap: Record<string, number> = {
    'Não tenho ainda': 0, '< R$5K': 0.5,
    'R$5K–R$15K': 1.5, 'R$15K–R$50K': 2.5, 'R$50K+': 3,
  }
  const budgetMap: Record<string, number> = {
    '< R$1K': 0, 'R$1K–R$3K': 0.5, 'R$3K–R$10K': 1, 'R$10K+': 1.5,
  }
  const reservaBonus = data.tem_reserva === 'Sim, tenho capital disponível' ? 0.5 : 0

  const financeiro = Math.min(4,
    (fatMap[data.faturamento_atual] ?? 0) +
    (budgetMap[data.budget_trafego] ?? 0) +
    reservaBonus
  )

  // ─── EXPERIÊNCIA (0–3) ─────────────────────────────────────
  const trafegoMap: Record<string, number> = {
    'Nunca rodei': 0, 'Já tentei / parei': 0.5, 'Sim, com frequência': 1,
  }
  const nivelNorm = (data.nivel_meta_ads - 1) / 4  // 0–1
  const funilBonus = data.usa_funil === 'Sim, completo' ? 0.5 : data.usa_funil === 'Parcial' ? 0.25 : 0
  const faturouBonus = data.ja_faturou_10k ? 0.5 : 0

  const experiencia = Math.min(3,
    (trafegoMap[data.rodou_trafego] ?? 0) +
    nivelNorm * 1.5 +
    funilBonus +
    faturouBonus
  )

  // ─── OPERACIONAL (0–2) ─────────────────────────────────────
  const pcMap: Record<string, number> = {
    'Só celular': 0, 'Sim, mas lento': 0.5, 'Sim, rápido': 1,
  }
  const horasMap: Record<string, number> = {
    '< 5h': 0, '5–10h': 0.5, '10–20h': 0.75, '20h+': 1,
  }

  const operacional = Math.min(2,
    (pcMap[data.tem_computador] ?? 0) +
    (horasMap[data.horas_semana] ?? 0)
  )

  // ─── COMPROMETIMENTO (0–1) ─────────────────────────────────
  const comprometimento = data.comprometimento === '100% comprometido' ? 1 : 0

  const breakdown: ScoreBreakdown = {
    financeiro: Math.round(financeiro * 10) / 10,
    experiencia: Math.round(experiencia * 10) / 10,
    operacional: Math.round(operacional * 10) / 10,
    comprometimento,
  }

  const total = Math.min(10, Math.round(
    breakdown.financeiro + breakdown.experiencia +
    breakdown.operacional + breakdown.comprometimento
  ))

  return { total, breakdown }
}

// Lead vai direto ao WhatsApp se score >= 6
export const SCORE_THRESHOLD_WA = 6
```

---

## Tela de resultado (pós-submit)

### Score >= 6 (qualificado)
```
✓  Você passou pela qualificação.

Cadu vai analisar suas respostas e, se o fit existir,
a conversa começa direto no WhatsApp.

[Falar no WhatsApp →]  (abre wa.me/5588999471567)
```

### Score < 6 (não qualificado)
```
Recebemos sua aplicação.

Analisaremos seu perfil e entraremos em contato
caso haja uma oportunidade de trabalhar juntos.

Obrigado pelo tempo dedicado.
```

O botão de WhatsApp só aparece para score >= `SCORE_THRESHOLD_WA`.
Nunca exibir o score bruto para o lead.
