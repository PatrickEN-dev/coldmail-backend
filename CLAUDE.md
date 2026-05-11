# CLAUDE.md — guia rápido pra Claude entrar no projeto Coldmail Backend

> Este arquivo é carregado automaticamente no início de cada sessão do Claude
> nesta pasta. Mantenha denso, factual e atualizado. Quando algo mudar
> arquiteturalmente, edite aqui também.

## Em uma frase

Backend NestJS que substitui, em 7 ondas (strangler fig), os 13 workflows
N8N do produto **Cold Email Pro**. Fonte da verdade do escopo:
[`nestjs-backend-brief.md`](./nestjs-backend-brief.md) (~1900 linhas).

## Stack

| Camada | Tecnologia | Notas |
|---|---|---|
| Runtime | Node.js 20 LTS | |
| Framework | NestJS 11 + Fastify 5 | `@nestjs/platform-fastify` |
| Linguagem | TypeScript 5.7 strict | paths `@/*`, `@infra/*`, `@modules/*`, `@shared/*` |
| ORM | **Prisma 6** (não Drizzle — override explícito do usuário) | `prisma/schema.prisma` alinhado com Supabase via MCP em 2026-05-11 |
| DB | Postgres do Supabase em prod, local em dev | RLS habilitada em todas as tabelas |
| Filas | BullMQ 5 + Redis (Railway) | 5 queues: dispatch, warmup, followups, schedules, webhooks |
| Cron | `@nestjs/schedule` | sempre passar `timeZone: 'America/Sao_Paulo'` em crons de negócio |
| Validação | Zod + `ZodValidationPipe` custom | DTOs em `src/modules/*/dto/*.dto.ts` |
| Auth | JWT Supabase HS256 verificado local via `jose` | `SUPABASE_JWT_SECRET`. `JwtAuthGuard` global via `APP_GUARD` |
| HTTP out | `undici` + `cockatiel` (retry + timeout + circuit breaker) | `ResilientEmailProvider` envolve cada strategy |
| AI | OpenAI direto (sem LangChain — decisão #8) | `OpenAiProvider` em `modules/ai/` |
| Logs | Pino + `nestjs-pino` com **redact agressivo** | secrets nunca aparecem em logs |
| Erros | Sentry opcional + 2 filters globais | `DomainExceptionFilter` + `SentryExceptionFilter` |
| Observabilidade | `prom-client` em `/metrics` (público) | counters customizados em `MetricsService` |
| Segurança | Helmet + Rate-Limit (300 req/min/IP) + Under-Pressure (503 sob carga) | tudo registrado em `main.ts` |
| Testes | Vitest | 22 testes (rate-limiter, parsers, pacing, clock, etc.) |

## Decisões importantes (override do brief)

| Tema | Decisão | Onde está documentado |
|---|---|---|
| ORM | Prisma 6 em vez de Drizzle | Decisão do usuário |
| Versões | Latest de tudo (NestJS 11, TS 5.7, Prisma 6) | Decisão do usuário |
| Warmup network | **Tenant-scoped** (pares só dentro do mesmo `userId`) | Decisão #16 → `warmup-pairing.ts` |
| Quota race fix | **Reserva atômica** (`updateMany WHERE today_usage < dailyLimit`) | `send-one.use-case.ts` |
| Resend rate-limit | RateLimiter local 5/s (legacy pt2 hit 429 — confirmado via MCP) | `resend.provider.ts` |
| Webhook HMAC | Usa **raw body** + opcional timestamp window 5min | `webhooks/webhooks.controller.ts` |
| SmartLead | **Só recebe webhooks**, não envia (Saga órfã no pt2 não migrada) | Decisão #4 |
| Edge Function `warmup-budget` | Mantida no Supabase | Decisão #5 |
| Timezone schedules | Via payload do front (column `tz` pendente — manual migration 001) | Decisão #9 |
| Workflows `[descarte]*` | Versões corretas já estão no Next.js (`/api/unipile-*`) | Decisão #11 |
| Warmup recv inbound | **IMAP polling** pra MVP; Pub/Sub doc'd em `docs/WARMUP_GMAIL_PUBSUB_MIGRATION.md` | Decisão #10 |

## Estrutura de pastas

```
src/
├── infra/
│   ├── cache/             Redis (ioredis)
│   ├── config/            Env Zod-validado + TypedConfigService
│   ├── database/          PrismaService (prefere DATABASE_POOL_URL)
│   ├── observability/     Pino + Sentry + Prometheus
│   └── queue/             5 BullMQ queues, constantes em `queue.constants.ts`
├── shared/
│   ├── domain/            Entity, ValueObject, AggregateRoot, DomainEvent
│   ├── errors/            DomainError + filter global
│   ├── events/            Catálogo de domain events (§7.4 do brief)
│   ├── http/              HMAC verify (timestamp window), rate-limiter, SSRF guard
│   └── pipes/             ZodValidationPipe
└── modules/
    ├── auth/              JwtAuthGuard global, @CurrentUser, @Public
    ├── health/            /health (Postgres + Redis)
    ├── leads/             CRUD `emails`
    ├── senders/           CRUD + cron reset-daily-usage (Onda 1)
    ├── templates/         CRUD + pickRandomVariant
    ├── schedules/         CRUD + ScheduleClock + cron tick (Onda 5)
    ├── dispatch/          POST /dispatch + worker (Onda 4)
    ├── providers/
    │   ├── email/         Strategy 7 implementations + ResilientEmailProvider
    │   └── linkedin/      UnipileProvider
    ├── webhooks/          Parsers + HMAC raw body + IngestEmailEventUseCase (Onda 2)
    ├── inbox/             email_messages thread API
    ├── warmup/            WarmupBudgetClient + pairing + crons (Onda 3 stub)
    ├── follow-ups/        Cron Mon-Thu 12h (Onda 6 stub)
    ├── ai/                OpenAI + prompts verbatim do brief
    ├── analytics/         Queries diretas (RPCs do Supabase usam auth.uid() que falha com service-role)
    ├── search/            POST /search (Onda 6 stub)
    └── linkedin/          Send DM/Invite + webhook Unipile (Onda 7 stub)
```

## Convenções OBRIGATÓRIAS

### Multi-tenant
- **Todo query novo precisa filtrar por `userId`**. Sem exceção.
- Exceções legítimas: `findLatestByEmailAny` (webhooks sem userId no payload) e crons globais (`resetAllDailyUsage`, `findDue` em schedules). Documente o motivo no comentário.
- `findByIdForUser(id, userId)`: padrão é `findUnique({ where: { id } })` + check `result.userId === userId`. Mais idiomático que `findFirst({ where: { id, userId } })`.
- Para `update` / `delete`, o serviço **sempre** chama `getByIdForUser` antes (validação de ownership). Repositórios usam `where: { id }`.

### Logs
- **Nunca logar API keys, JWTs, senhas, refresh tokens.** Pino redact (`logger.module.ts`) já cobre uma lista grande, mas se adicionar novo campo sensível, atualize lá também.
- Use Pino logger (não `console.log`) exceto em `main.ts` antes do bootstrap completar.

### Webhooks
- Inbound webhooks são `@Public()`, **mas** validam HMAC.
- Use `req.rawBody` (não `JSON.stringify(body)`) na verificação HMAC — formatação JSON quebra assinatura.
- `verifyHmacSignature` em `shared/http/webhook-signature.ts` aceita opcional `timestamp` + `toleranceSec` (default 300s) para anti-replay.
- Idempotency via `email_messages.unique(provider_message_id, direction)` — partial index, NULLs ignorados.

### Crons
- Sempre passar `timeZone: 'America/Sao_Paulo'` em crons de negócio (envio, follow-up, reset).
- Crons que processam jobs devem ser idempotentes (a outra réplica pode rodar o mesmo tick).

### Providers de email
- Constructor **nunca** lança — se config falta, `client` fica null e `send()` que lança.
- Sempre passe pelo `EmailProviderRegistry.resolve()` (ele envolve em `ResilientEmailProvider`).
- Resend tem rate-limit 5/s já dentro da strategy.

### Validação
- Todo body de endpoint usa `ZodValidationPipe(...Schema)`. Todo `:id` usa `ParseUUIDPipe`.
- DTOs ficam em `src/modules/*/dto/*.dto.ts`.

### Migrations
- **Local**: `npx prisma migrate dev --name xxx` cria diretório em `prisma/migrations/`.
- **Prod (Supabase)**: aplicar via `mcp__supabase__apply_migration` (idempotente). SQL fica em `prisma/manual-migrations/`.
- DEPOIS de aplicar em prod, rodar `npx prisma db pull` localmente pra reconciliar.

## Schema Prisma — gotchas conhecidas

| Campo | Sutil |
|---|---|
| `emails.user_id` | **Nullable** (legacy). Webhooks dropam evento se for null. |
| `emails.reply_time` | É **text** em prod, não timestamptz. Salvar como `.toISOString()`. |
| `emails.lead_name` | Nome do lead (não `name`). |
| `emails.campaign_name` | (não `campaign`). |
| `schedules.type` | (não `scheduleType`). |
| `schedules.scheduled_date` | Tipo Postgres `date` (não string). Converter via `new Date('YYYY-MM-DDT00:00:00Z')`. |
| `schedules.scheduled_time` | Tipo Postgres `time(6)`. Converter via `new Date('1970-01-01THH:MM:00Z')`. |
| `schedules.tz` | **Não existe ainda**. Hardcoded `America/Sao_Paulo`. Migration manual `001` pendente. |
| `schedules.lead_selections` | jsonb (não `payload`). |
| `linkedin_accounts.user_id` | **Nullable** (legacy). Backfill manual: `prisma/manual-migrations/003`. As 2 rows existentes em prod estão NULL. |
| `linkedin_accounts.data_conecction` | Typo preservado de prod (não `data_connection`). |
| `sender_emails.unique` | Composto `(user_id, email_address)`. Use `findByAddressForUser`. |
| `email_messages` | **Tabela criada por esta migração** (002). Já aplicada em prod 2026-05-11. |

## Workflows N8N → módulos Nest (mapeamento)

| Workflow N8N | Módulo Nest | Status |
|---|---|---|
| `GIFZ8zzIWiXGdral` pt1 Split | `DispatchModule.SendBatchUseCase` (entry payload §6.2.1) | ✅ |
| `jhzBrpA2g5mYOMon` pt2 Send | `DispatchModule.SendOneUseCase` + ResilientEmailProvider | ✅ |
| `NkZO6yq9LeKVBnbs` Webhook eventos | `WebhooksController` + `IngestEmailEventUseCase` | ✅ |
| `8CamkGMPY06aiLQ7` SmartLead | `SmartLeadWebhookParser` | ✅ |
| `GaDxY8f5dQnP0LG4` Warmup envio | `WarmupModule.WarmupTickCron` (stub) | 🟡 |
| `bTuTALx2EDDqBrxK` Warmup recv | `WarmupModule` (stub IMAP) | 🟡 |
| `0x9tjMCXLxba1LqZ` Follow-ups | `FollowUpsTickCron` (stub) | 🟡 |
| `G1G1DkHf7GrU79us` zera limite | `SendersModule.ResetDailyUsageCron` | ✅ |
| `estBS0PmeL1hFpDe` LinkedIn send | `LinkedInController.send` | 🟡 |
| `8FaGelWVDKyoAS7r` LinkedIn events | `LinkedInController` webhook | 🟡 |
| `nNEGPw9Eb4suATn3` Pesquisa V1 | `SearchController` (stub) | 🟡 |
| `UBXSpTG6kyijdzaw` descarte LinkedIn auth | **Fica no Next.js** (decisão #11) | ✅ |
| `6hgCvqOmiAoFjQG7` descarte Unipile callback | **Fica no Next.js** (decisão #11) | ✅ |

Workflows N8N **não-CEP** (Telegram, Jessica, etc.): **ignorar** — pertencem a outros projetos.

## Comandos úteis

```bash
npm run start:dev          # dev com watch
npm run start              # roda sem watch
npm run build              # nest build (SWC)
npm run typecheck          # tsc --noEmit
npm run lint               # eslint --fix
npm test                   # vitest run

npm run prisma:generate    # gera @prisma/client
npm run prisma:migrate:dev # nova migration local
npm run prisma:pull        # introspect Supabase (use após apply em prod)
npm run prisma:studio      # UI de browsing
```

## MCPs disponíveis nesta sessão

- `mcp__supabase__*` — `list_tables`, `execute_sql`, `apply_migration`, `list_migrations`, `get_advisors`, etc.
- `mcp__n8n-mcp__*` — `n8n_list_workflows`, `n8n_get_workflow`, `n8n_executions`, etc.

Use pra:
- Conferir drift `schema.prisma ↔ Supabase` quando algo parecer fora.
- Investigar erros em execuções N8N (decisão #2 do brief).
- Aplicar DDL em prod (cuidado — sempre confirmar com o usuário antes).

## Anti-patterns a evitar

- ❌ **Não** rodar DDL em prod sem confirmar com o usuário primeiro.
- ❌ **Não** commitar `.env` (já no `.gitignore`, mas confirmar antes de `git add`).
- ❌ **Não** chamar `console.log` em código de runtime — use Pino.
- ❌ **Não** usar `JSON.stringify(body)` em verificação HMAC — use `req.rawBody`.
- ❌ **Não** retornar `Email` direto com `displayName: null` em rotas REST sem ajustar — UI espera `string`.
- ❌ **Não** adicionar enums Prisma novos — o DB usa text + CHECK constraint. Use Zod no DTO.
- ❌ **Não** logar `provider_metadata` ou qualquer campo `*.secret*` / `*.token*` / `*.api_key*` — Pino redact cobre, mas ampliar lista quando novo campo aparecer.
- ❌ **Não** misturar `findUnique({ where: { id, userId } })` — Prisma só aceita unique no where do update. Padrão é findUnique por PK + check de tenant.

## O que está intencionalmente NÃO feito

- Warmup worker BullMQ funcional (Onda 3 stub)
- Follow-up worker funcional (Onda 6 stub)
- Search com pipeline RapidAPI completa (Onda 6 stub)
- LinkedIn DM/invite com persistência completa em `linkedin_messages` (Onda 7 stub)
- Pub/Sub do Gmail (escolha B = IMAP polling, mas doc completo de Pub/Sub em `docs/WARMUP_GMAIL_PUBSUB_MIGRATION.{md,html}` pra futuro)
- LangChain (uso OpenAI puro — decisão #8)
- GraphQL/tRPC (REST + Zod é suficiente)
- Microservices (monolito modular)
- Event sourcing (domain events só pra desacoplar módulos)
- `dispatch_outbox` table (decisão #7 — postergar até problema real)

## Arquivos críticos pra ler primeiro

1. `src/main.ts` — bootstrap (security stack)
2. `src/app.module.ts` — modules
3. `src/modules/dispatch/application/send-one.use-case.ts` — coração da Onda 4
4. `src/modules/webhooks/application/ingest-email-event.use-case.ts` — coração da Onda 2
5. `src/modules/schedules/application/fire-schedule.use-case.ts` — bridge Onda 5 → 4
6. `prisma/schema.prisma` — schema vivo (alinhado com Supabase prod)
7. `nestjs-backend-brief.md` — fonte do escopo (sempre consultar)

## Quando NÃO ter certeza

- Estado real do Supabase: rode `mcp__supabase__list_tables({verbose: true})`
- Estado real dos workflows N8N: rode `mcp__n8n-mcp__n8n_list_workflows({active: true})`
- Estado real de uma execução: `mcp__n8n-mcp__n8n_executions({action: 'get', id, mode: 'error'})`

## Convenções de comunicação com o usuário

- **pt-BR** sempre na conversa.
- Código, comentários, logs, mensagens de commit: **inglês**.
- Antes de ação destrutiva (DDL prod, force push, delete cascade): **confirme**.
