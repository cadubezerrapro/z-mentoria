# mentoria-backend

> Sistema de qualificação de leads + painel CRM para a mentoria individual de Cadu Bezerra.

---

## O que é este projeto

Antes de o lead chegar no WhatsApp do Cadu, ele passa por um formulário de qualificação
de 6 etapas. Os dados são salvos num banco PostgreSQL no Railway, um score 0–10 é calculado
automaticamente e o lead só recebe o link do WA se atingir o threshold mínimo.

O painel CRM (acesso privado) permite visualizar, filtrar, comentar e evoluir o status de
cada lead em tempo real.

---

## Repositório

```
mentoria-backend/          ← raiz do projeto Next.js
├── app/                   ← App Router (Next.js 14)
├── components/
├── lib/
├── prisma/
├── docs/backend/          ← esta pasta de documentação
│   ├── README.md          ← você está aqui
│   ├── ARCHITECTURE.md    ← diagrama, fluxo, decisões técnicas
│   ├── DATABASE.md        ← schema Prisma + migrations + seed
│   ├── FORM.md            ← especificação completa do formulário
│   ├── CRM.md             ← painel admin: features, rotas, componentes
│   ├── API.md             ← contratos de todos os endpoints
│   ├── SECURITY.md        ← todas as camadas de segurança
│   ├── DEPLOY.md          ← deploy Railway passo a passo
│   └── ENV.md             ← variáveis de ambiente, regras, como gerar
└── .env.example
```

---

## Início rápido (desenvolvimento local)

```bash
# 1. Clonar e instalar
git clone https://github.com/cadubezerra/mentoria-backend
cd mentoria-backend
npm install

# 2. Copiar e preencher variáveis
cp .env.example .env.local

# 3. Subir banco local com Docker
docker compose up -d

# 4. Migrations + seed
npx prisma migrate dev
npx prisma db seed

# 5. Rodar
npm run dev
```

URLs locais:
- Formulário: `http://localhost:3000/aplicar`
- CRM: `http://localhost:3000/crm`
- API: `http://localhost:3000/api`

---

## Links rápidos

| Doc | Descrição |
|-----|-----------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Diagrama, fluxo completo, stack e decisões |
| [DATABASE.md](./DATABASE.md) | Schema Prisma, índices, migrations, seed |
| [FORM.md](./FORM.md) | Todas as etapas, campos, validações, score |
| [CRM.md](./CRM.md) | Painel admin, rotas, componentes, status |
| [API.md](./API.md) | Todos os endpoints, contratos, exemplos |
| [SECURITY.md](./SECURITY.md) | Rate limit, CSP, LGPD, auth, honeypot |
| [DEPLOY.md](./DEPLOY.md) | Railway setup, env vars, CI/CD, domínio |
| [ENV.md](./ENV.md) | `.env.example` completo, regras, como gerar secrets |
