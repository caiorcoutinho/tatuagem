# Research: Agendamento para Tatuadores e Estúdios

**Feature**: `001-agendamento-tatuagem` | **Date**: 2026-11-12 | **Plan**: [plan.md](./plan.md)

Stack definida pelo usuário: **React + FastAPI, PostgreSQL, MinIO, nginx na frente**, com UI
responsiva para mobile e desktop. Este documento registra as decisões que completam essa
stack e resolvem os pontos em aberto do Technical Context.

---

## R1. Backend: FastAPI + camada de dados

- **Decision**: Python 3.12, FastAPI, Pydantic v2 (+ pydantic-settings), SQLAlchemy 2.0
  assíncrono com asyncpg, Alembic para migrações. Servido por Uvicorn.
- **Rationale**: SQLAlchemy 2.0 é o ORM padrão do ecossistema FastAPI e permite descer para
  SQL quando necessário (constraints de exclusão, advisory locks — ver R4). Alembic dá
  migrações versionadas e revisáveis.
- **Alternatives considered**: SQLModel (abstração extra sobre SQLAlchemy, sem ganho real
  aqui); Tortoise ORM (ecossistema menor, migrações menos maduras).

## R2. Frontend: React responsivo

- **Decision**: React 19 + TypeScript + Vite (SPA), React Router, TanStack Query para estado
  de servidor, React Hook Form + Zod em formulários, Tailwind CSS mobile-first. Tipos da API
  gerados do OpenAPI do FastAPI com `openapi-typescript`.
- **Rationale**: Tailwind com breakpoints mobile-first atende "responsivo para mobile e
  computadores" sem CSS próprio extenso. Tipos gerados do OpenAPI eliminam duplicação de
  contratos entre back e front (Princípio I). TanStack Query resolve cache, loading e erro
  sem store global.
- **Alternatives considered**: Next.js (descartado: usuário definiu FastAPI como backend);
  Redux (desnecessário — quase todo o estado é de servidor); MUI/Chakra (mais peso, menos
  controle visual para um produto centrado em imagens).
- **Grade de portfólio (estilo mural)**: CSS `columns` (masonry nativo) com imagens em
  dimensões conhecidas (largura/altura salvas no upload) para evitar layout shift; lazy
  loading nativo (`loading="lazy"`). Sem biblioteca de masonry.

## R3. Autenticação e sessão

- **Decision**: e-mail + senha com hash Argon2 (`argon2-cffi`). Sessão server-side: token
  opaco aleatório em cookie `HttpOnly; Secure; SameSite=Lax`, com hash do token salvo na
  tabela `session` (expira em 30 dias, renovável). Frontend e API no mesmo domínio via nginx,
  então não há CORS.
- **CSRF**: mutações exigem `Content-Type: application/json` (ou multipart para upload) e
  validação do header `Origin` contra o domínio configurado.
- **Autorização**: dependências FastAPI por papel (`require_role(STUDIO)`) e checagens de
  posse no serviço (ex.: só o tatuador do agendamento aprova).
- **Rationale**: sessão opaca é revogável (logout, troca de senha), evita armazenar JWT no
  browser e é simples com mesmo domínio.
- **Alternatives considered**: JWT stateless (revogação difícil, sem benefício com um único
  backend); FastAPI-Users (traz mais do que o necessário e acopla o modelo de usuário).

## R4. Concorrência: capacidade e sobreposição

- **Decision**:
  1. **Sobreposição do tatuador (FR-010)**: constraint de exclusão no PostgreSQL com
     `btree_gist`: `EXCLUDE USING gist (artist_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
     WHERE (status IN ('pending','confirmed'))`. O banco garante a regra entre estúdios,
     mesmo com requisições simultâneas.
  2. **Capacidade do horário (FR-007a)**: na transação de criação, `pg_advisory_xact_lock`
     com chave derivada de `(studio_id, starts_at)`, contagem dos agendamentos ativos e
     comparação com a capacidade da ocorrência.
  3. Violação de qualquer um → `409 Conflict` com código de erro específico
     (`artist_unavailable`, `slot_full`).
- **Rationale**: atende SC-003 (0% de sobreposição) sem depender de checagens na aplicação,
  que falham sob concorrência. Advisory lock serializa só o horário disputado.
- **Alternatives considered**: isolamento SERIALIZABLE em tudo (retries frequentes, mais
  complexo); lock otimista com versão (exige linha materializada por horário — ver R5).

## R5. Modelo de horários: padrão semanal + exceções

- **Decision**: ocorrências **calculadas sob demanda**, não materializadas:
  - `weekly_rule` (dia da semana, hora início/fim no fuso do estúdio, capacidade);
  - `date_override` (data específica: bloqueada, ou faixas/capacidade substitutas);
  - `one_off_slot` (horário avulso extra com início/fim/capacidade).
  O serviço de disponibilidade gera as ocorrências de um intervalo (máx. 60 dias) aplicando
  regras → overrides → avulsos e subtrai agendamentos ativos. O agendamento guarda
  `starts_at`/`ends_at` (snapshot), então mudanças no padrão não afetam agendamentos já
  feitos (edge case da spec).
- **Rationale**: atende FR-007b sem job de geração de horários nem limpeza de linhas futuras;
  alterar o padrão é uma edição simples.
- **Alternatives considered**: materializar ocorrências para N semanas (precisa de job
  recorrente, migração em massa ao editar o padrão e reconciliação com agendamentos).
- **Fuso horário**: cada estúdio tem `timezone` IANA; regras em hora local, conversão com
  `zoneinfo` (stdlib); armazenamento em UTC (`timestamptz`).

## R6. Expiração de pendentes e envio de e-mails

- **Decision**: um processo **worker** (mesma imagem do backend, comando diferente no
  compose) roda em loop a cada 60 s:
  1. marca como `expired` os agendamentos `pending` com `expires_at <= now()` e cria as
     notificações;
  2. envia e-mails das notificações com `email_sent_at IS NULL` (padrão outbox), com até 5
     tentativas.
  `expires_at = min(requested_at + 48h, starts_at)` é calculado na criação. Além disso, a
  criação de agendamento expira, na mesma transação, pendentes vencidos do mesmo
  tatuador/horário, para que a constraint de exclusão não bloqueie por pendente já vencido.
- **Rationale**: notificação gravada na mesma transação do evento não se perde se o SMTP
  falhar (FR-012). Um único worker simples evita fila externa.
- **Alternatives considered**: Celery/RQ + Redis (infra extra sem necessidade no volume da
  v1); `BackgroundTasks` do FastAPI (perde e-mails em restart); pg_cron (acopla regra de
  negócio ao banco).
- **E-mail**: SMTP via `aiosmtplib`; em desenvolvimento, Mailpit no compose.

## R7. Imagens do portfólio com MinIO

- **Decision**: upload multipart pela API (`POST /api/portfolio/items`), limite 10 MB,
  formatos JPEG/PNG/WebP. O backend valida com Pillow (tipo real, não só extensão; remove
  EXIF), gera duas variantes WebP — `thumb` (largura 600 px) e `full` (largura máx. 1600 px)
  — e grava no bucket `portfolio` do MinIO via `boto3` (API S3). O bucket tem política de
  leitura anônima e é servido pelo nginx em `/media/`.
- **Rationale**: thumbnails pequenos garantem a meta de SC-004 (grade com 100 trabalhos em
  < 2 s); validação no servidor impede upload de arquivo arbitrário; leitura via nginx tira a
  carga de imagens do backend.
- **Alternatives considered**: URL pré-assinada de upload direto ao MinIO (não permite
  validar/redimensionar antes de publicar sem um segundo passo assíncrono); servir imagens
  pela API (gargalo desnecessário).

## R8. nginx

- **Decision**: nginx como único ponto de entrada:
  - `/` → build estático do frontend com fallback para `index.html` (SPA);
  - `/api/` → backend (Uvicorn);
  - `/media/` → MinIO (bucket `portfolio`, somente GET, cache longo — chaves imutáveis).
  Limite `client_max_body_size 10m`, gzip, cabeçalhos de segurança, TLS no ambiente de
  produção.
- **Rationale**: mesmo domínio para tudo (cookies de sessão simples, sem CORS) e cache de
  estáticos/imagens na borda.

## R9. Busca (FR-017)

- **Decision**: consultas SQL com filtros: cidade (igualdade normalizada), estilo (join por
  tabela de estilos) e nome (`ILIKE` com índice trigram `pg_trgm`). Paginação por cursor.
- **Rationale**: volume da v1 não justifica motor de busca dedicado.
- **Alternatives considered**: Elasticsearch/Meilisearch (infra extra, YAGNI).

## R10. Link de acesso do visitante (FR-002)

- **Decision**: ao criar um agendamento como visitante, gera token aleatório de 32 bytes;
  só o hash SHA-256 é salvo no agendamento. O e-mail traz
  `/agendamentos/acesso/{token}`, que permite ver e cancelar aquele agendamento. Ao criar
  conta Cliente com e-mail verificado, agendamentos com o mesmo e-mail são associados
  (FR-002a).
- **Rationale**: o token vazado não compromete outros agendamentos; associar só após
  verificação do e-mail impede que alguém "tome" agendamentos de outra pessoa.
- **Consequência**: contas Cliente exigem verificação de e-mail antes da associação.

## R11. Qualidade e testes (Constituição)

- **Decision**:
  - Backend: `ruff` (lint + format), `mypy --strict` nos módulos de domínio, `pytest` +
    `pytest-asyncio` + `httpx.AsyncClient`; testes de integração contra PostgreSQL real
    (container do compose de teste).
  - Frontend: ESLint + Prettier, Vitest + Testing Library; Playwright para e2e em dois
    viewports (mobile 390×844 e desktop 1440×900).
  - Repositório: `pre-commit` com hooks de ruff, eslint/prettier e
    `conventional-pre-commit` (validação de commits Angular); GitHub Actions rodando lint,
    testes e e2e.
- **Rationale**: cobre os itens obrigatórios de "Padrões de Qualidade" (lint/format no
  pre-commit e CI, validação de mensagem de commit, testes de regra de negócio).

## R12. Ambiente e deploy

- **Decision**: Docker Compose com serviços `nginx`, `frontend` (build → estático servido
  pelo nginx), `backend`, `worker`, `postgres` (16), `minio`, `mailpit` (apenas dev).
  Configuração por variáveis de ambiente; segredos fora do repositório (`.env` no
  `.gitignore`, `.env.example` versionado).
- **Rationale**: mesmo ambiente para dev, CI e um servidor único de produção na v1.
