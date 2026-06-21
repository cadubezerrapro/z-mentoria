# Testes

Stack de testes: **Vitest** (unit + integração) + **Playwright** (E2E).
Nenhuma dependência de banco real nos unit tests — só nos de integração.

---

## Instalação

```bash
npm install -D vitest @vitest/coverage-v8 @testing-library/react @testing-library/user-event
npm install -D supertest @types/supertest
npm install -D @playwright/test
npx playwright install chromium
```

`package.json`:

```json
{
  "scripts": {
    "test":         "vitest run",
    "test:watch":   "vitest",
    "test:coverage":"vitest run --coverage",
    "test:e2e":     "playwright test",
    "test:e2e:ui":  "playwright test --ui"
  }
}
```

`vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config'
import path from 'path'

export default defineConfig({
  test: {
    environment: 'node',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      include: ['lib/**', 'app/api/**'],
      thresholds: { lines: 80, functions: 80, branches: 75 },
    },
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, '.') },
  },
})
```

---

## Estrutura de pastas

```
tests/
├── unit/
│   ├── score.test.ts        ← calcularScore() — coração do negócio
│   ├── sanitize.test.ts
│   └── validations/
│       └── lead.schema.test.ts
├── integration/
│   ├── api.leads.test.ts    ← POST /api/leads
│   ├── api.crm.test.ts      ← GET/PATCH/DELETE autenticados
│   └── helpers/
│       ├── db.ts            ← setup/teardown banco de teste
│       └── fixtures.ts      ← payloads prontos
└── e2e/
    ├── form.spec.ts         ← fluxo completo do formulário
    └── crm.spec.ts          ← login + tabela de leads

tests/setup.ts               ← mocks globais (Upstash, Prisma)
playwright.config.ts
```

---

## Unit Tests

### `score.test.ts` — `calcularScore()`

O score é a lógica de negócio mais crítica. Cada subcomponente tem testes isolados.

```typescript
import { describe, it, expect } from 'vitest'
import { calcularScore, SCORE_THRESHOLD_WA } from '@/lib/score'
import type { LeadFormData } from '@/lib/validations/lead.schema'

const base: LeadFormData = {
  nome: 'Teste Lead',
  whatsapp: '(88) 99999-0000',
  email: 'teste@email.com',
  comoEncontrou: 'Instagram',
  oque_vende: 'Infoproduto',
  segmento: 'Saúde',
  tempo_mercado: '1–3 anos',
  vende_internacional: 'Só BR',
  faturamento_atual: 'R$15K–R$50K',
  budget_trafego: 'R$3K–R$10K',
  tem_reserva: 'Sim, tenho capital disponível',
  fonte_vendas: 'Misto',
  rodou_trafego: 'Sim, com frequência',
  nivel_meta_ads: 4,
  ja_faturou_10k: true,
  usa_funil: 'Sim, completo',
  ferramentas: ['Hotmart'],
  tem_computador: 'Sim, rápido',
  horas_semana: '10–20h',
  tem_cnpj: 'Sim',
  tem_equipe: 'Não, só eu',
  maior_dificuldade: 'Não consigo escalar sem perder margem',
  por_que_agora: 'Quero faturar em dólar este ano',
  meta_90dias: 'R$100K',
  comprometimento: '100% comprometido',
  consentimento_lgpd: true,
  honeypot: '',
}

describe('calcularScore — financeiro (max 4)', () => {
  it('faturamento R$50K+ + budget R$10K+ + reserva = 5 → clampa em 4', () => {
    const { breakdown } = calcularScore({
      ...base,
      faturamento_atual: 'R$50K+',
      budget_trafego: 'R$10K+',
      tem_reserva: 'Sim, tenho capital disponível',
    })
    expect(breakdown.financeiro).toBe(4)
  })

  it('faturamento < R$5K + sem reserva + budget < R$1K = 0.5', () => {
    const { breakdown } = calcularScore({
      ...base,
      faturamento_atual: '< R$5K',
      budget_trafego: '< R$1K',
      tem_reserva: 'Não tenho',
    })
    expect(breakdown.financeiro).toBe(0.5)
  })

  it('"Não tenho ainda" fatura = 0 no eixo financeiro base', () => {
    const { breakdown } = calcularScore({
      ...base,
      faturamento_atual: 'Não tenho ainda',
      budget_trafego: '< R$1K',
      tem_reserva: 'Não tenho',
    })
    expect(breakdown.financeiro).toBe(0)
  })
})

describe('calcularScore — experiência (max 3)', () => {
  it('tráfego frequente + nivel 5 + funil completo + ja faturou 10k = 3 (clamped)', () => {
    const { breakdown } = calcularScore({
      ...base,
      rodou_trafego: 'Sim, com frequência',
      nivel_meta_ads: 5,
      usa_funil: 'Sim, completo',
      ja_faturou_10k: true,
    })
    expect(breakdown.experiencia).toBe(3)
  })

  it('nunca rodou tráfego + nivel 1 + sem funil + nunca faturou 10k = 0', () => {
    const { breakdown } = calcularScore({
      ...base,
      rodou_trafego: 'Nunca rodei',
      nivel_meta_ads: 1,
      usa_funil: 'Não uso',
      ja_faturou_10k: false,
    })
    expect(breakdown.experiencia).toBe(0)
  })
})

describe('calcularScore — operacional (max 2)', () => {
  it('PC rápido + 20h+ por semana = 2', () => {
    const { breakdown } = calcularScore({
      ...base,
      tem_computador: 'Sim, rápido',
      horas_semana: '20h+',
    })
    expect(breakdown.operacional).toBe(2)
  })

  it('só celular + menos de 5h = 0', () => {
    const { breakdown } = calcularScore({
      ...base,
      tem_computador: 'Só celular',
      horas_semana: '< 5h',
    })
    expect(breakdown.operacional).toBe(0)
  })
})

describe('calcularScore — comprometimento (0 ou 1)', () => {
  it('"100% comprometido" = 1', () => {
    const { breakdown } = calcularScore({ ...base, comprometimento: '100% comprometido' })
    expect(breakdown.comprometimento).toBe(1)
  })

  it('"Depende do valor" = 0', () => {
    const { breakdown } = calcularScore({ ...base, comprometimento: 'Depende do valor' })
    expect(breakdown.comprometimento).toBe(0)
  })
})

describe('calcularScore — total e threshold', () => {
  it('lead ideal → total >= SCORE_THRESHOLD_WA', () => {
    const { total } = calcularScore(base) // base já é o lead ideal
    expect(total).toBeGreaterThanOrEqual(SCORE_THRESHOLD_WA)
  })

  it('lead fraco (tudo mínimo) → total < SCORE_THRESHOLD_WA', () => {
    const { total } = calcularScore({
      ...base,
      faturamento_atual: 'Não tenho ainda',
      budget_trafego: '< R$1K',
      tem_reserva: 'Não tenho',
      rodou_trafego: 'Nunca rodei',
      nivel_meta_ads: 1,
      ja_faturou_10k: false,
      usa_funil: 'Não uso',
      tem_computador: 'Só celular',
      horas_semana: '< 5h',
      comprometimento: 'Ainda avaliando',
    })
    expect(total).toBeLessThan(SCORE_THRESHOLD_WA)
  })

  it('total nunca ultrapassa 10', () => {
    const { total } = calcularScore({
      ...base,
      faturamento_atual: 'R$50K+',
      budget_trafego: 'R$10K+',
      tem_reserva: 'Sim, tenho capital disponível',
      rodou_trafego: 'Sim, com frequência',
      nivel_meta_ads: 5,
      usa_funil: 'Sim, completo',
      ja_faturou_10k: true,
      tem_computador: 'Sim, rápido',
      horas_semana: '20h+',
      comprometimento: '100% comprometido',
    })
    expect(total).toBeLessThanOrEqual(10)
  })

  it('SCORE_THRESHOLD_WA é 6', () => {
    expect(SCORE_THRESHOLD_WA).toBe(6)
  })
})
```

---

### `lead.schema.test.ts` — Validação Zod

```typescript
import { describe, it, expect } from 'vitest'
import { leadSchema } from '@/lib/validations/lead.schema'

const valid = {
  nome: 'João Silva',
  whatsapp: '(88) 99999-4567',
  email: 'joao@email.com',
  // ... demais campos com valores válidos
}

describe('leadSchema — validação', () => {
  it('aceita payload completo válido', () => {
    expect(leadSchema.safeParse(valid).success).toBe(true)
  })

  it('rejeita WhatsApp sem DDD', () => {
    const r = leadSchema.safeParse({ ...valid, whatsapp: '99999-4567' })
    expect(r.success).toBe(false)
    if (!r.success) expect(r.error.issues[0].path).toContain('whatsapp')
  })

  it('rejeita email inválido', () => {
    const r = leadSchema.safeParse({ ...valid, email: 'nao-e-email' })
    expect(r.success).toBe(false)
  })

  it('rejeita consentimento_lgpd: false', () => {
    const r = leadSchema.safeParse({ ...valid, consentimento_lgpd: false })
    expect(r.success).toBe(false)
  })

  it('rejeita nivel_meta_ads fora de 1–5', () => {
    const r = leadSchema.safeParse({ ...valid, nivel_meta_ads: 6 })
    expect(r.success).toBe(false)
  })
})
```

---

### `sanitize.test.ts`

```typescript
import { describe, it, expect } from 'vitest'
import { sanitizeText } from '@/lib/sanitize'

describe('sanitizeText', () => {
  it('remove tags HTML', () => {
    expect(sanitizeText('<script>alert(1)</script>texto')).toBe('texto')
  })

  it('remove atributos maliciosos', () => {
    expect(sanitizeText('<img src onerror="alert(1)">texto')).toBe('texto')
  })

  it('faz trim', () => {
    expect(sanitizeText('  oi  ')).toBe('oi')
  })

  it('preserva texto sem HTML', () => {
    expect(sanitizeText('Texto normal com acentos: ção')).toBe('Texto normal com acentos: ção')
  })
})
```

---

## Integration Tests

> Requerem banco PostgreSQL rodando. Use Docker via `docker compose up -d` antes de rodar.
> Variáveis: `.env.test` com `DATABASE_URL` apontando para banco de teste separado.

### `tests/integration/helpers/fixtures.ts`

```typescript
export const leadQualificado = {
  nome: 'Maria Qualificada',
  whatsapp: '(11) 99999-1234',
  email: 'maria@email.com',
  comoEncontrou: 'Instagram',
  oque_vende: 'Curso de finanças',
  segmento: 'Finanças',
  tempo_mercado: '1–3 anos',
  vende_internacional: 'BR + Internacional',
  faturamento_atual: 'R$15K–R$50K',
  budget_trafego: 'R$3K–R$10K',
  tem_reserva: 'Sim, tenho capital disponível',
  fonte_vendas: 'Misto',
  rodou_trafego: 'Sim, com frequência',
  nivel_meta_ads: 4,
  ja_faturou_10k: true,
  usa_funil: 'Sim, completo',
  ferramentas: ['Hotmart'],
  tem_computador: 'Sim, rápido',
  horas_semana: '10–20h',
  tem_cnpj: 'Sim',
  tem_equipe: 'Não, só eu',
  maior_dificuldade: 'Preciso de orientação para escalar internacionalmente',
  por_que_agora: 'Quero faturar em dólar nos próximos 3 meses',
  meta_90dias: 'R$100K',
  comprometimento: '100% comprometido',
  consentimento_lgpd: true,
  honeypot: '',
  utm_source: 'instagram',
  utm_medium: 'cpc',
  utm_campaign: 'test',
}

export const leadFraco = {
  ...leadQualificado,
  faturamento_atual: 'Não tenho ainda',
  budget_trafego: '< R$1K',
  tem_reserva: 'Não tenho',
  rodou_trafego: 'Nunca rodei',
  nivel_meta_ads: 1,
  ja_faturou_10k: false,
  usa_funil: 'Não uso',
  tem_computador: 'Só celular',
  horas_semana: '< 5h',
  comprometimento: 'Ainda avaliando',
}
```

### `api.leads.test.ts`

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import { prisma } from '@/lib/prisma'
import { leadQualificado, leadFraco } from './helpers/fixtures'

const BASE = 'http://localhost:3000'

async function post(path: string, body: object) {
  return fetch(`${BASE}${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  })
}

afterAll(async () => {
  await prisma.lead.deleteMany({ where: { utm_campaign: 'test' } })
})

describe('POST /api/leads', () => {
  it('201 + redirectToWhatsApp:true para lead qualificado', async () => {
    const res = await post('/api/leads', leadQualificado)
    const data = await res.json()
    expect(res.status).toBe(201)
    expect(data.success).toBe(true)
    expect(data.score).toBeGreaterThanOrEqual(6)
    expect(data.redirectToWhatsApp).toBe(true)
  })

  it('201 + redirectToWhatsApp:false para lead fraco', async () => {
    const res = await post('/api/leads', leadFraco)
    const data = await res.json()
    expect(res.status).toBe(201)
    expect(data.redirectToWhatsApp).toBe(false)
  })

  it('400 com campo inválido (email malformado)', async () => {
    const res = await post('/api/leads', { ...leadQualificado, email: 'invalido' })
    expect(res.status).toBe(400)
    const data = await res.json()
    expect(data.fields).toHaveProperty('email')
  })

  it('200 fake quando honeypot está preenchido', async () => {
    const res = await post('/api/leads', { ...leadQualificado, honeypot: 'bot' })
    const data = await res.json()
    expect(res.status).toBe(200)
    expect(data.success).toBe(true)
    expect(data.redirectToWhatsApp).toBe(false)
    // confirma que não criou no banco
    const count = await prisma.lead.count({ where: { email: leadQualificado.email } })
    expect(count).toBe(0)
  })

  it('salva lead no banco após submissão válida', async () => {
    const uniqueEmail = `test+${Date.now()}@email.com`
    await post('/api/leads', { ...leadQualificado, email: uniqueEmail })
    const lead = await prisma.lead.findFirst({ where: { email: uniqueEmail } })
    expect(lead).not.toBeNull()
    expect(lead?.score).toBeGreaterThanOrEqual(0)
  })

  it('429 após 5 requisições no mesmo IP', async () => {
    // Nota: rate limit testado com IP de test isolado via header
    const headers = { 'Content-Type': 'application/json', 'x-forwarded-for': '10.0.0.254' }
    for (let i = 0; i < 5; i++) {
      await fetch(`${BASE}/api/leads`, {
        method: 'POST',
        headers,
        body: JSON.stringify(leadFraco),
      })
    }
    const res = await fetch(`${BASE}/api/leads`, {
      method: 'POST',
      headers,
      body: JSON.stringify(leadFraco),
    })
    expect(res.status).toBe(429)
  })
})

describe('GET /api/leads (sem auth)', () => {
  it('redireciona para /login sem sessão', async () => {
    const res = await fetch(`${BASE}/api/leads`)
    expect([302, 307, 401]).toContain(res.status)
  })
})
```

---

## E2E Tests (Playwright)

`playwright.config.ts`:

```typescript
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  use: {
    baseURL: 'http://localhost:3000',
    headless: true,
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  webServer: {
    command: 'npm run dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
})
```

### `form.spec.ts` — Fluxo completo do formulário

```typescript
import { test, expect } from '@playwright/test'

test.describe('Formulário de qualificação', () => {
  test('lead qualificado → redirect para WhatsApp na tela de resultado', async ({ page }) => {
    await page.goto('/aplicar')

    // Step 1 — Identificação
    await page.fill('[name="nome"]', 'Ana Teste')
    await page.fill('[name="whatsapp"]', '(88) 99999-0001')
    await page.fill('[name="email"]', `ana+${Date.now()}@email.com`)
    await page.click('text=Próximo')

    // Step 2 — Negócio
    await page.fill('[name="oque_vende"]', 'Mentoria de copywriting')
    await page.click('label:has-text("Finanças")')
    await page.click('label:has-text("1–3 anos")')
    await page.click('label:has-text("BR + Internacional")')
    await page.click('text=Próximo')

    // Step 3 — Financeiro
    await page.click('label:has-text("R$15K–R$50K")')
    await page.click('label:has-text("R$3K–R$10K")')
    await page.click('label:has-text("Sim, tenho capital disponível")')
    await page.click('label:has-text("Misto")')
    await page.click('text=Próximo')

    // Step 4 — Experiência
    await page.click('label:has-text("Sim, com frequência")')
    await page.click('label:has-text("Já faturou > R$10K")')
    await page.click('label:has-text("Sim, completo")')
    await page.click('label:has-text("Hotmart")')
    await page.click('text=Próximo')

    // Step 5 — Operacional
    await page.click('label:has-text("Sim, rápido")')
    await page.click('label:has-text("10–20h")')
    await page.click('label:has-text("Sim")')
    await page.click('label:has-text("Não, só eu")')
    await page.click('text=Próximo')

    // Step 6 — Motivação
    await page.fill('[name="maior_dificuldade"]', 'Preciso escalar sem perder margem no tráfego pago')
    await page.fill('[name="por_que_agora"]', 'Quero faturar em dólar nos próximos meses')
    await page.click('label:has-text("R$100K")')
    await page.click('label:has-text("100% comprometido")')
    await page.check('[name="consentimento_lgpd"]')
    await page.click('text=Enviar aplicação')

    // Tela de resultado
    await page.waitForURL('**/obrigado')
    await expect(page.locator('text=Falar no WhatsApp')).toBeVisible()
  })

  test('step 1 — campo WhatsApp sem DDD exibe erro de validação', async ({ page }) => {
    await page.goto('/aplicar')
    await page.fill('[name="whatsapp"]', '99999-4567') // sem DDD
    await page.click('text=Próximo')
    await expect(page.locator('text=Formato: (88) 99999-9999')).toBeVisible()
    await expect(page.url()).toContain('/aplicar') // não avançou
  })

  test('step 6 — botão Enviar desabilitado sem aceite LGPD', async ({ page }) => {
    // Preencher steps 1–5 antes
    // ... (reutilizar helper de preenchimento)
    const btn = page.locator('button:has-text("Enviar aplicação")')
    await expect(btn).toBeDisabled()
  })

  test('barra de progresso avança a cada step', async ({ page }) => {
    await page.goto('/aplicar')
    const progress = page.locator('[role="progressbar"]')
    const initial = await progress.getAttribute('aria-valuenow')

    // avançar para step 2
    await page.fill('[name="nome"]', 'Teste')
    await page.fill('[name="whatsapp"]', '(88) 99999-0002')
    await page.fill('[name="email"]', 'teste2@email.com')
    await page.click('text=Próximo')

    const after = await progress.getAttribute('aria-valuenow')
    expect(Number(after)).toBeGreaterThan(Number(initial))
  })
})
```

### `crm.spec.ts` — Login e tabela de leads

```typescript
import { test, expect } from '@playwright/test'

const ADMIN_EMAIL = process.env.TEST_ADMIN_EMAIL ?? 'admin@mentoria.com'
const ADMIN_PASS  = process.env.TEST_ADMIN_PASS  ?? 'senhatest123'

test.describe('CRM — autenticação', () => {
  test('rota /crm sem login redireciona para /login', async ({ page }) => {
    await page.goto('/crm')
    await expect(page).toHaveURL(/\/login/)
  })

  test('login com credenciais válidas acessa o CRM', async ({ page }) => {
    await page.goto('/login')
    await page.fill('[name="email"]', ADMIN_EMAIL)
    await page.fill('[name="password"]', ADMIN_PASS)
    await page.click('[type="submit"]')
    await expect(page).toHaveURL(/\/crm/)
    await expect(page.locator('h1, [data-testid="crm-title"]')).toBeVisible()
  })

  test('login com senha errada exibe mensagem de erro', async ({ page }) => {
    await page.goto('/login')
    await page.fill('[name="email"]', ADMIN_EMAIL)
    await page.fill('[name="password"]', 'senhaerrada')
    await page.click('[type="submit"]')
    await expect(page.locator('text=Credenciais inválidas')).toBeVisible()
  })
})
```

---

## CI — GitHub Actions

`.github/workflows/test.yml`:

```yaml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm test
      - run: npm run test:coverage

  e2e:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: mentoria_test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 5s --health-timeout 3s --health-retries 5
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/mentoria_test
      NEXTAUTH_SECRET: secret-de-test-com-32-chars-minimo
      NEXTAUTH_URL: http://localhost:3000
      UPSTASH_REDIS_REST_URL: ${{ secrets.UPSTASH_REDIS_REST_URL }}
      UPSTASH_REDIS_REST_TOKEN: ${{ secrets.UPSTASH_REDIS_REST_TOKEN }}
      IP_SALT: salt-de-test
      TEST_ADMIN_EMAIL: ${{ secrets.TEST_ADMIN_EMAIL }}
      TEST_ADMIN_PASS: ${{ secrets.TEST_ADMIN_PASS }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx prisma migrate deploy
      - run: npx prisma db seed
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## Checklist de testes pré-deploy

```
[ ] npm test         — todos os unit tests passando
[ ] npm run test:coverage — cobertura ≥ 80% em lib/score.ts
[ ] npm run test:e2e — fluxo completo do form passando
[ ] Rate limit: 5 POSTs rápidos → 6º retorna 429
[ ] Honeypot: campo preenchido → 200 sem criar lead no banco
[ ] Score limítrofe: lead com score 5 não recebe link WA; score 6 recebe
[ ] LGPD: DELETE /api/leads/[id]/gdpr → campos anonimizados no banco
[ ] Auth: /crm sem login → redirect /login; com login → acesso ok
[ ] Lockout: 6 logins errados → conta bloqueada 15 min
```
