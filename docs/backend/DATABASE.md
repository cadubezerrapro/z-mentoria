# Banco de Dados

Banco: **PostgreSQL** provisionado pelo Railway.
ORM: **Prisma** com migrations versionadas.

---

## Schema completo (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─────────────────────────────────────────────
// LEAD
// ─────────────────────────────────────────────

model Lead {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // ETAPA 1 — Identificação
  nome          String
  whatsapp      String
  email         String
  comoEncontrou String? // "Instagram" | "Indicação" | "Google" | "Outro"

  // ETAPA 2 — Negócio
  oque_vende          String   @db.VarChar(200)
  segmento            String   // enum-like: "Saúde" | "Finanças" | ...
  tempo_mercado       String   // "< 6 meses" | "6m–1ano" | "1–3 anos" | "3+ anos"
  vende_internacional String   // "Só BR" | "BR + Internacional" | "Só Internacional"

  // ETAPA 3 — Financeiro
  faturamento_atual String // "< R$5K" | "R$5K–R$15K" | "R$15K–R$50K" | "R$50K+" | "Não tenho"
  budget_trafego    String // "< R$1K" | "R$1K–R$3K" | "R$3K–R$10K" | "R$10K+"
  tem_reserva       String // "Sim, tenho" | "Estou juntando" | "Não tenho"
  fonte_vendas      String // "Orgânico" | "Tráfego pago" | "Indicação" | "Lançamento" | "Misto"

  // ETAPA 4 — Experiência
  rodou_trafego  String  // "Sim, frequente" | "Já tentei" | "Nunca"
  nivel_meta_ads Int     // 1–5
  ja_faturou_10k Boolean
  usa_funil      String  // "Sim, completo" | "Parcial" | "Não uso"
  ferramentas    String[]

  // ETAPA 5 — Operacional
  tem_computador String // "Sim, rápido" | "Sim, lento" | "Só celular"
  horas_semana   String // "< 5h" | "5–10h" | "10–20h" | "20h+"
  tem_cnpj       String // "Sim" | "Não, só PF" | "Em processo"
  tem_equipe     String // "Sim" | "Não, só eu" | "Terceirizado"

  // ETAPA 6 — Motivação
  maior_dificuldade String  @db.VarChar(300)
  por_que_agora     String  @db.VarChar(300)
  meta_90dias       String  // "R$10K" | "R$30K" | "R$50K" | "R$100K" | "R$100K+"
  comprometimento   String  // "100% comprometido" | "Avaliando" | "Depende do valor"

  // METADATA DE TRÁFEGO
  ip_hash          String? // SHA-256(IP) — nunca o IP em claro
  user_agent_hash  String?
  utm_source       String?
  utm_medium       String?
  utm_campaign     String?
  referrer         String?
  honeypot_trigger Boolean @default(false) // true = bot, não processar

  // QUALIFICAÇÃO
  score           Int   @default(0) // 0–10
  score_breakdown Json?
  // ex: { "financeiro": 3, "experiencia": 2.5, "operacional": 2, "comprometimento": 1 }

  // CRM
  status        LeadStatus         @default(NOVO)
  notas_internas String?           @db.Text
  statusHistory  LeadStatusHistory[]

  // LGPD
  consentimento_lgpd Boolean  @default(false)
  apagado_em         DateTime? // preenchido pelo endpoint GDPR DELETE

  @@index([status])
  @@index([score])
  @@index([createdAt])
  @@index([honeypot_trigger])
  @@index([email]) // busca por email no CRM
}

// ─────────────────────────────────────────────
// HISTÓRICO DE STATUS
// ─────────────────────────────────────────────

model LeadStatusHistory {
  id        String     @id @default(cuid())
  leadId    String
  lead      Lead       @relation(fields: [leadId], references: [id], onDelete: Cascade)
  status    LeadStatus
  changedAt DateTime   @default(now())
  nota      String?    // nota opcional ao mudar status
  changedBy String?    // email do admin que mudou

  @@index([leadId])
}

// ─────────────────────────────────────────────
// ADMIN (CRM)
// ─────────────────────────────────────────────

model AdminUser {
  id           String    @id @default(cuid())
  email        String    @unique
  passwordHash String    // bcrypt, salt 12
  name         String?
  createdAt    DateTime  @default(now())
  lastLoginAt  DateTime?
  failedLogins Int       @default(0) // para lockout
  lockedUntil  DateTime? // lockout após 5 falhas
}

// ─────────────────────────────────────────────
// ENUM
// ─────────────────────────────────────────────

enum LeadStatus {
  NOVO
  EM_ANALISE
  QUALIFICADO
  NAO_QUALIFICADO
  EM_NEGOCIACAO
  CONVERTIDO
  PERDIDO
}
```

---

## Índices e performance

| Índice | Campo | Motivo |
|--------|-------|--------|
| `status` | Lead.status | Filtro mais usado no CRM |
| `score` | Lead.score | Ordenação e filtro por score |
| `createdAt` | Lead.createdAt | Ordenação padrão (mais recente) |
| `email` | Lead.email | Busca por email no CRM |
| `honeypot_trigger` | Lead.honeypot_trigger | Excluir bots da listagem |
| `leadId` | LeadStatusHistory.leadId | JOIN no histórico |

---

## Migrations

```bash
# Criar migration (desenvolvimento)
npx prisma migrate dev --name create_leads

# Aplicar em produção (Railway)
npx prisma migrate deploy

# Inspecionar banco atual
npx prisma studio

# Reset completo (só dev)
npx prisma migrate reset
```

**Regra:** nunca editar arquivos dentro de `prisma/migrations/` à mão.
Sempre criar nova migration para qualquer alteração de schema.

---

## Seed (`prisma/seed.ts`)

```typescript
import { PrismaClient } from '@prisma/client'
import bcrypt from 'bcrypt'

const prisma = new PrismaClient()

async function main() {
  const email = process.env.ADMIN_EMAIL
  const password = process.env.ADMIN_INITIAL_PASSWORD

  if (!email || !password) {
    throw new Error('ADMIN_EMAIL e ADMIN_INITIAL_PASSWORD devem estar no .env')
  }

  const existing = await prisma.adminUser.findUnique({ where: { email } })
  if (existing) {
    console.log('Admin já existe. Seed ignorado.')
    return
  }

  const passwordHash = await bcrypt.hash(password, 12)

  await prisma.adminUser.create({
    data: { email, passwordHash, name: 'Cadu Bezerra' }
  })

  console.log(`Admin criado: ${email}`)
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect())
```

Executar uma única vez após deploy inicial:
```bash
railway run npx prisma db seed
```

---

## Singleton Prisma (`lib/prisma.ts`)

```typescript
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'error', 'warn']
      : ['error'],
  })

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

O singleton evita múltiplas conexões abertas em desenvolvimento com hot-reload.

---

## Retention policy (LGPD)

Leads com status `PERDIDO` ou `NAO_QUALIFICADO` são apagados após 90 dias.
Implementado como cron job no Railway (ou GitHub Actions scheduled):

```typescript
// scripts/cleanup-old-leads.ts
import { prisma } from '../lib/prisma'

const RETENTION_DAYS = 90
const cutoff = new Date()
cutoff.setDate(cutoff.getDate() - RETENTION_DAYS)

await prisma.lead.deleteMany({
  where: {
    status: { in: ['PERDIDO', 'NAO_QUALIFICADO'] },
    updatedAt: { lt: cutoff },
  }
})
```
