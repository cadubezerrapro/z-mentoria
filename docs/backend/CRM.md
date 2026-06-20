# Painel CRM

Painel privado acessado em `crm.dominio.com.br`.
Protegido por NextAuth — sem sessão válida, middleware redireciona para `/login`.
UI: **shadcn/ui + 21st.dev**, tema dark, tipografia Inter.

---

## Rotas do CRM

| Rota | Componente | Descrição |
|------|-----------|-----------|
| `/login` | `LoginPage` | Tela de autenticação |
| `/crm` | `DashboardPage` | Visão geral + gráficos |
| `/crm/leads` | `LeadsPage` | Tabela filtrada de leads |
| `/crm/leads/[id]` | `LeadDetailPage` | Detalhe completo do lead |

---

## Layout do CRM (`components/crm/layout.tsx`)

```
┌─────────────────────────────────────────────────────────┐
│ SIDEBAR (240px, dark #0a0a0a)                           │
│  ┌─────────────┐                                        │
│  │   Logo      │                                        │
│  └─────────────┘                                        │
│  ● Dashboard                                            │
│  ● Leads        [badge: 12 novos]                       │
│  ──────────                                             │
│  ○ Sair                                                 │
│                                                         │
│                MAIN CONTENT                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Header: título da página + user badge           │   │
│  ├─────────────────────────────────────────────────┤   │
│  │  Conteúdo da página                              │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Dashboard (`/crm`)

### Grid de métricas (`StatsGrid.tsx`)

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│  Total Leads │  Esta semana │  Qualificados│  Convertidos │
│     247      │      18      │      89      │      23      │
│  ↑ +12 hoje  │              │   (score≥6)  │  taxa 25.8%  │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

Cards shadcn/ui com ícone Lucide, número grande, subtexto de variação.

### Gráfico de leads por dia (`LeadsChart.tsx`)

- Lib: **Recharts** (Área + Linha)
- Período: últimos 30 dias (selector: 7d / 30d / 90d)
- Cor: `#22c55e` (verde da identidade)
- Tooltip customizado com data + quantidade

### Distribuição de scores (`ScoreDistribution.tsx`)

- BarChart horizontal: score 0–10 × quantidade de leads
- Barras coloridas: vermelho (0–4) · amarelo (5–7) · verde (8–10)

---

## Lista de Leads (`/crm/leads`)

### Filtros (`LeadFilters.tsx`)

```
[Buscar por nome, email ou WA ___________]

Status:        [Todos ▼]
Score mínimo:  [0 ▼]
Faturamento:   [Todos ▼]
Segmento:      [Todos ▼]
Período:       [Últimos 30d ▼]

[Limpar filtros]   [Exportar CSV]
```

### Tabela (`LeadTable.tsx`)

Componente `DataTable` do shadcn/ui com paginação server-side.

| Coluna | Conteúdo |
|--------|---------|
| Nome | string |
| WhatsApp | masked: `(88) 9••••-4567` |
| Score | `ScoreBadge`: `9/10 ●` (verde/amarelo/vermelho) |
| Faturamento | label do enum |
| Status | `StatusBadge` colorido |
| Cadastro | data relativa: "há 2 horas" |
| Ações | `→` link para detalhe |

Ordenação por coluna (score, data, status).
Paginação: 20 por página, query param `?page=`.

### ScoreBadge (`components/crm/leads/ScoreBadge.tsx`)

```
score 0–4  →  fundo vermelho escuro  #450a0a  texto #fca5a5
score 5–6  →  fundo amarelo escuro   #422006  texto #fcd34d
score 7–8  →  fundo verde escuro     #052e16  texto #4ade80
score 9–10 →  fundo verde puro       #15803d  texto #fff   + borda brilhante
```

---

## Detalhe do Lead (`/crm/leads/[id]`)

### Header (`LeadHeader.tsx`)

```
┌────────────────────────────────────────────────────────┐
│  João Silva                              Score: 9/10 ● │
│  (88) 99999-4567 · joao@email.com                      │
│  Cadastrado em 20/06/2026 às 14:32                     │
│                                                        │
│  [Abrir WhatsApp ↗]  [Mudar Status ▼]  [Apagar]       │
└────────────────────────────────────────────────────────┘
```

### Grid de dados (`LeadDataGrid.tsx`)

Cards colapsáveis por seção (Accordion shadcn/ui):

```
▼ Identificação
  Nome: João Silva
  WhatsApp: (88) 99999-4567
  Email: joao@email.com
  Como encontrou: Instagram

▼ Negócio
  O que vende: Mentoria de emagrecimento
  Segmento: Saúde
  Tempo de mercado: 1–3 anos
  Mercado: BR + Internacional

▼ Financeiro
  Faturamento atual: R$15K–R$50K
  Budget tráfego: R$3K–R$10K
  Tem reserva: Sim, tenho capital disponível
  Fonte de vendas: Misto

▼ Experiência
  Rodou tráfego: Sim, com frequência
  Nível Meta Ads: 4/5
  Já faturou R$10K+: Sim
  Usa funil: Sim, completo
  Ferramentas: Hotmart, Manychat

▼ Operacional
  Computador: Sim, rápido
  Horas/semana: 10–20h
  CNPJ: Sim
  Equipe: Não, só eu

▼ Motivação
  Maior dificuldade: "Não consigo escalar o tráfego..."
  Por que agora: "Quero faturar em dólar antes de..."
  Meta 90 dias: R$50K
  Comprometimento: 100% comprometido
```

### Score breakdown (`ScoreBreakdown.tsx`)

```
Financeiro:      ████████░░  3.5/4
Experiência:     ███████░░░  2.8/3
Operacional:     ████████░░  1.75/2
Comprometimento: ██████████  1/1
─────────────────────────────────
Total:           ████████░░  9/10
```

Progress bars coloridas (shadcn/ui Progress component).

### Timeline de status (`StatusTimeline.tsx`)

```
● CONVERTIDO          21/06/2026 15:10
  "Pagamento confirmado, acesso liberado"

● EM_NEGOCIACAO       20/06/2026 18:30
  "Chamada agendada para amanhã"

● QUALIFICADO         20/06/2026 14:45
  (status automático por score)

● NOVO                20/06/2026 14:32
  (registro inicial)
```

### Notas internas (`NotasEditor.tsx`)

```
┌────────────────────────────────────────┐
│  Notas sobre este lead                 │
│                                        │
│  [textarea editável]                   │
│  "Falou em chamada que tem um..."      │
│                                        │
│                              [Salvar]  │
└────────────────────────────────────────┘
```

Auto-save com debounce de 1.5s após parar de digitar.
`PATCH /api/leads/[id]` com `{ notas_internas: "..." }`.

---

## Status — Fluxo e cores

```
NOVO  →  EM_ANALISE  →  QUALIFICADO  →  EM_NEGOCIACAO  →  CONVERTIDO
                     ↘  NAO_QUALIFICADO                 ↘  PERDIDO
```

| Status | Cor fundo | Cor texto |
|--------|-----------|-----------|
| NOVO | `#1c1c1c` | `#9ca3af` |
| EM_ANALISE | `#422006` | `#fcd34d` |
| QUALIFICADO | `#052e16` | `#4ade80` |
| NAO_QUALIFICADO | `#450a0a` | `#fca5a5` |
| EM_NEGOCIACAO | `#1e3a5f` | `#93c5fd` |
| CONVERTIDO | `#14532d` | `#bbf7d0` |
| PERDIDO | `#1c0808` | `#f87171` |

---

## Export CSV (`ExportButton.tsx`)

```typescript
// GET /api/leads/export (auth obrigatório)
// Retorna text/csv com headers corretos para download

// Campos exportados:
// id, nome, whatsapp, email, score, status,
// faturamento_atual, segmento, createdAt
// WA mascarado: nunca exportar em claro sem confirmação
```

---

## Login (`/login`)

```
┌────────────────────────────────┐
│           Logo                 │
│                                │
│  E-mail                        │
│  [________________________]    │
│                                │
│  Senha                         │
│  [________________________]    │
│                                │
│  [         Entrar         ]    │
│                                │
│  "Acesso restrito"             │
└────────────────────────────────┘
```

- Sem link de "esqueci a senha" público (segurança)
- Recuperação: reset direto no banco via Railway CLI
- Após 5 tentativas erradas: lockout 15 min (Redis)
- Sessão JWT expira em 8h (NextAuth `maxAge: 28800`)
