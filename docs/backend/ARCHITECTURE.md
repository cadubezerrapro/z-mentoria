# Arquitetura

## Stack de decisão

| Camada | Escolha | Alternativa descartada | Motivo |
|--------|---------|----------------------|--------|
| Framework | **Next.js 14 (App Router)** | Express separado | API Routes + SSR + React no mesmo processo; zero infra extra |
| Banco | **PostgreSQL (Railway)** | Supabase | Railway já provisiona PG junto com o deploy; sem conta extra |
| ORM | **Prisma** | Drizzle | Migrations CLI maduras, type-safety gerada automaticamente |
| Auth CRM | **NextAuth v5 (Credentials)** | Clerk | Sem custo, sem dependência externa, total controle |
| Rate limit | **Upstash Redis** | Vercel KV | Grátis até 10 k req/dia, SDK nativo com `@upstash/ratelimit` |
| UI | **shadcn/ui + 21st.dev** | MUI / Chakra | Sem bundle gordo; componentes copiados no projeto, customizáveis |
| Email | **Resend** | SendGrid | SDK simples, free tier 3 k emails/mês, alta entregabilidade |
| Formulário | **React Hook Form + Zod** | Formik | Performance (sem re-renders), validação compartilhada com backend |
| Deploy | **Railway** | Vercel + Supabase | PostgreSQL + Node.js num só projeto, preço previsível |

---

## Diagrama de fluxo completo

```
┌─────────────────────────────────────────────────────────────┐
│  index.html  (LP estática – Python/Apache)                  │
│                                                             │
│  [Quero me aplicar]  ──────────────────────────────────┐   │
└────────────────────────────────────────────────────────┼───┘
                                                         │  redirect
                                                         ▼
┌─────────────────────────────────────────────────────────────┐
│  aplicar.dominio.com.br   (Next.js – Railway)               │
│                                                             │
│  Step 1 → Step 2 → Step 3 → Step 4 → Step 5 → Step 6       │
│                                                             │
│  [Submit]                                                   │
│    │                                                        │
│    ├─ honeypot check   ──── preenchido? → 200 fake, ignora  │
│    ├─ rate limit check ──── excedeu?   → 429, mensagem      │
│    ├─ zod validation   ──── inválido?  → 400 + campo        │
│    ├─ sanitização                                           │
│    ├─ calcularScore()  ──── 0–10                            │
│    ├─ prisma.lead.create()                                  │
│    └─ notificar (email Resend)                              │
│                                                             │
│  [Tela de Resultado]                                        │
│    ├─ score >= 6  → "Aprovado — ir para WhatsApp"           │
│    └─ score < 6   → "Entraremos em contato"                 │
└─────────────────────────────────────────────────────────────┘
                         │
                         │ salva no banco
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Railway PostgreSQL                                         │
│  Lead, LeadStatusHistory, AdminUser                         │
└──────────────────────────────┬──────────────────────────────┘
                               │ Prisma queries
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  crm.dominio.com.br   (Next.js – mesmo Railway service)     │
│                                                             │
│  /crm          Dashboard + gráficos                         │
│  /crm/leads    Tabela filtrada de leads                     │
│  /crm/leads/[id]  Detalhe + notas + status                 │
│                                                             │
│  Auth: NextAuth Credentials → JWT session → middleware      │
└─────────────────────────────────────────────────────────────┘
```

---

## Estrutura de pastas do projeto

```
mentoria-backend/
│
├── app/
│   ├── (public)/                   # rotas públicas (sem auth)
│   │   ├── aplicar/
│   │   │   ├── page.tsx            # formulário multi-step
│   │   │   ├── loading.tsx
│   │   │   └── obrigado/
│   │   │       └── page.tsx        # tela de resultado pós-submit
│   │   └── layout.tsx              # layout minimalista (sem sidebar)
│   │
│   ├── (crm)/                      # rotas protegidas
│   │   ├── crm/
│   │   │   ├── page.tsx            # /crm  → dashboard
│   │   │   ├── leads/
│   │   │   │   ├── page.tsx        # lista de leads + filtros
│   │   │   │   └── [id]/
│   │   │   │       └── page.tsx    # detalhe do lead
│   │   │   └── layout.tsx          # sidebar + header do CRM
│   │   └── login/
│   │       └── page.tsx            # tela de login admin
│   │
│   ├── api/
│   │   ├── leads/
│   │   │   ├── route.ts            # POST /api/leads (público)
│   │   │   │                       # GET  /api/leads (auth)
│   │   │   └── [id]/
│   │   │       ├── route.ts        # PATCH + DELETE (auth)
│   │   │       └── gdpr/
│   │   │           └── route.ts    # DELETE completo LGPD
│   │   ├── dashboard/
│   │   │   └── route.ts            # GET métricas (auth)
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts
│   │
│   ├── layout.tsx                  # root layout (fontes, providers)
│   └── globals.css
│
├── components/
│   ├── form/
│   │   ├── MultiStepForm.tsx       # orquestra steps + estado global
│   │   ├── StepIndicator.tsx       # barra de progresso
│   │   ├── steps/
│   │   │   ├── Step1Identificacao.tsx
│   │   │   ├── Step2Negocio.tsx
│   │   │   ├── Step3Financeiro.tsx
│   │   │   ├── Step4Experiencia.tsx
│   │   │   ├── Step5Operacional.tsx
│   │   │   └── Step6Motivacao.tsx
│   │   └── ResultScreen.tsx
│   │
│   ├── crm/
│   │   ├── Sidebar.tsx
│   │   ├── Header.tsx
│   │   ├── dashboard/
│   │   │   ├── StatsGrid.tsx       # 4 cards de métricas
│   │   │   ├── LeadsChart.tsx      # gráfico de linha (Recharts)
│   │   │   └── ScoreDistribution.tsx
│   │   ├── leads/
│   │   │   ├── LeadTable.tsx
│   │   │   ├── LeadFilters.tsx
│   │   │   ├── LeadRow.tsx
│   │   │   └── ExportButton.tsx
│   │   └── lead-detail/
│   │       ├── LeadHeader.tsx
│   │       ├── LeadDataGrid.tsx
│   │       ├── StatusTimeline.tsx
│   │       ├── NotasEditor.tsx
│   │       └── ActionButtons.tsx
│   │
│   └── ui/                         # shadcn/ui copiados + customizados
│
├── lib/
│   ├── prisma.ts                   # singleton PrismaClient
│   ├── auth.ts                     # NextAuth config + callbacks
│   ├── ratelimit.ts                # Upstash Ratelimit instance
│   ├── score.ts                    # calcularScore() + threshold
│   ├── notify.ts                   # sendLeadEmail() via Resend
│   ├── sanitize.ts                 # sanitizeText() wrapper
│   └── validations/
│       ├── lead.schema.ts          # Zod schema completo do formulário
│       └── auth.schema.ts          # Zod schema de login
│
├── prisma/
│   ├── schema.prisma
│   ├── migrations/                 # geradas automaticamente
│   └── seed.ts                     # cria AdminUser inicial
│
├── middleware.ts                   # protege /crm/* e /api/leads GET/PATCH
├── next.config.js                  # headers segurança + rewrites
├── docker-compose.yml              # PG local para desenvolvimento
├── .env.example
├── .gitignore
└── package.json
```

---

## Decisões de design

**Por que um único Next.js para form + CRM?**
Simplifica o deploy (um serviço Railway), compartilha as libs Prisma/Zod/auth,
e evita CORS entre front e API. Em escala maior isso seria separado.

**Por que não usar Vercel?**
Vercel não tem banco relacional nativo. Usaríamos Supabase + Vercel, que são
duas contas, dois dashboards, dois billing. Railway resolve tudo num só lugar.

**Por que shadcn/ui + 21st.dev e não um design system pronto?**
Os componentes são copiados no projeto — sem dependência de versão de terceiro.
Customização total, bundle menor, sem breaking changes de lib externa.
