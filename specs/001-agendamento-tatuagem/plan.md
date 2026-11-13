# Implementation Plan: Agendamento para Tatuadores e Estúdios

**Branch**: `001-agendamento-tatuagem` | **Date**: 2026-11-13 (revisão 3: conta = pessoa,
equipe do estúdio) | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-agendamento-tatuagem/spec.md`

## Summary

Aplicação web responsiva (mobile e desktop) para agendamento de sessões de tatuagem. Cada
**conta é uma pessoa**: pode ter perfil de tatuador, ser dona de um estúdio, ser membro da
equipe de vários estúdios e agendar como cliente; visitantes também agendam sem conta. O
estúdio é gerido pelo dono e pela equipe (membros), com registro de todas as ações de
gestão. Ele informa as macas, publica horários (padrão semanal, exceções e avulsos), define
os dias em que cada tatuador vinculado atende e a janela de antecedência. A soma de sessões
simultâneas nunca passa das macas.

Todo agendamento exige estúdio + tatuador publicáveis (e-mail verificado) e com vínculo
ativo, traz descrição, local, tamanho e até 3 imagens de referência privadas, nasce
pendente, é aprovado pelo tatuador, expira em 48h sem resposta, pode ser remarcado pelo
cliente e cancelado até o início; cada cliente tem no máximo 2 pedidos pendentes. Há
lembrete ao cliente 24h antes e resumo diário ao tatuador. Convites de vínculo expiram em
24h. O tatuador publica um portfólio em grade. A v1 inclui verificação de e-mail,
recuperação de senha, exclusão de conta e de estúdio e anonimização após 12 meses (LGPD).
Backup fica fora da v1 (risco aceito na spec).

Abordagem técnica: SPA **React** + API **FastAPI**, **PostgreSQL** (concorrência garantida
no banco por constraint de exclusão e advisory locks por estúdio e por contato do cliente),
**autorização por recurso** (dono/membro de cada estúdio, perfil de tatuador, parte do
agendamento) em vez de papel fixo na conta, **MinIO** com bucket público (portfólio) e
privado (referências), **nginx** na frente servindo o frontend, a API e as imagens públicas
no mesmo domínio, e **mensageria assíncrona com outbox transacional + RabbitMQ + Celery**:
a API grava a mudança de negócio e a mensagem de outbox na mesma transação; um relay publica
no RabbitMQ; workers Celery enviam os e-mails (com retentativa e dead-letter); o Celery Beat
dispara as periódicas (expirar pedidos e convites, concluir sessões, lembretes, resumo
diário, anonimização, limpeza do registro). Detalhes e alternativas em [research.md](./research.md).

## Technical Context

**Language/Version**: Python 3.12 (backend); TypeScript 5 + React 19 (frontend)

**Primary Dependencies**: FastAPI, python-multipart, Pydantic v2, SQLAlchemy 2.0 (async) +
asyncpg, Alembic, argon2-cffi, boto3 (MinIO), Pillow, aiosmtplib, Celery 5 (broker
RabbitMQ via kombu/py-amqp) · Vite, React Router,
TanStack Query, React Hook Form + Zod + @hookform/resolvers, Tailwind CSS,
openapi-typescript · @axe-core/playwright (testes). Justificativas em research.md.

**Storage**: PostgreSQL 16 (`citext`, `btree_gist`, `pg_trgm`; também a tabela de outbox);
MinIO (bucket público `portfolio`, bucket privado `booking-refs`); RabbitMQ 3.13 (filas
duráveis `email`, `default`, `email.dlq`)

**Testing**: pytest + pytest-asyncio + httpx (integração com Postgres real, relógio
controlado para jobs); Vitest + Testing Library; Playwright (e2e em 390×844 e 1440×900) + axe

**Target Platform**: navegadores modernos, mobile e desktop; servidor Linux com Docker Compose
(nginx, backend, outbox-relay, celery-worker, celery-beat, rabbitmq, postgres, minio;
mailpit em dev)

**Project Type**: aplicação web (frontend SPA + backend API + relay de outbox + workers
Celery)

**Performance Goals**: portfólio com 100 trabalhos visível em < 2 s (SC-004); fluxo de
agendamento concluído em < 3 min (SC-001)

**Constraints**: 0% de sobreposição de agendamentos, 0% de excesso de macas e no máximo 2
pendentes por contato sob concorrência (SC-003, FR-007c, FR-011c); toda ação de gestão do
estúdio auditada na mesma transação (FR-023); remoção de membro com efeito imediato
(FR-021); upload ≤ 10 MB por imagem; horários no fuso do estúdio; UI utilizável a partir de
360 px; LGPD: contato, detalhes e imagens restritos às partes e anonimizados após 12 meses

**Scale/Scope**: v1 para centenas de estúdios, alguns milhares de tatuadores e dezenas de
milhares de agendamentos/mês; cerca de 30 telas; servidor único

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio | Verificação | Status |
|---|---|---|
| I. Qualidade de Código | ruff + mypy (backend), ESLint + Prettier (frontend), testes obrigatórios de regra de negócio, CI bloqueando merge (R11); processamento de imagem e validação de agendamento compartilhados, sem duplicação | ✅ |
| II. Código Enxuto | Toda dependência externa tem motivo em research.md; Celery + RabbitMQ justificados em R6 (retentativa com backoff, dead-letter, escala de consumidores e periódicas, por decisão do usuário); sem motor de busca, lib de masonry ou framework de permissões | ✅ |
| III. SOLID | Módulos de domínio; regras puras em `domain/`; autorização isolada em dependências (`studio_access`, `require_artist`, `booking_party`) e não espalhada pelos routers; auditoria num serviço único (`AuditTrail`); `Protocol`s para relógio, storage, e-mail e providers entre módulos | ✅ |
| IV. Clean Code | Nomes de domínio da spec (`booking`, `affiliation`, `studio_member`, `seats`, `weekdays`); erros com códigos explícitos; no máximo 3 parâmetros, com objetos de parâmetros (`ScheduleSnapshot`, `DateRange`, `NotificationRequest`, `AuditEntry`) | ✅ |
| V. YAGNI | Só dois níveis de acesso (dono, membro) sem RBAC genérico; ocorrências calculadas; um único mecanismo assíncrono (outbox → RabbitMQ → Celery) para e-mails e periódicas; `Protocol` só com duas implementações; sem backup automatizado (fora do escopo pela spec) | ✅ |
| VI. Commits Semânticos | `conventional-pre-commit` no hook `commit-msg` e checagem no CI | ✅ |
| Padrões de Qualidade | pre-commit local + CI; `.env` fora do git com `.env.example` | ✅ |

**Gate (pré-pesquisa)**: aprovado, sem violações.

**Re-check pós-design (revisão 3, 2026-11-13)**: aprovado, sem dependência nova. A troca de
"perfil único por conta" por "conta = pessoa" (research.md R19) **simplifica** a
autorização: um único mecanismo por recurso substitui papéis fixos e já cobre dono, membro,
tatuador e cliente. Equipe e registro de ações (R20), dias de atendimento e janela (R21) e
lembretes (R22) reaproveitam transação, outbox e tasks Celery. A troca do worker caseiro
por Celery + RabbitMQ (R6, decisão do usuário) mantém a atomicidade pelo outbox
transacional e reaproveita os casos de uso assíncronos nas tasks (`asyncio.run`), sem
camada síncrona duplicada. Revisões anteriores
corrigiram os achados D1, D2 e F3 do `/speckit-analyze`. Nenhuma entrada em Complexity
Tracking é necessária.

## Project Structure

### Documentation (this feature)

```text
specs/001-agendamento-tatuagem/
├── plan.md              # Este arquivo
├── research.md          # Phase 0: decisões técnicas (R1–R22)
├── data-model.md        # Phase 1: entidades, regras e transições
├── quickstart.md        # Phase 1: guia de validação ponta a ponta (V1–V14)
├── contracts/
│   ├── api.md           # Contrato REST /api/v1
│   └── ui-routes.md     # Rotas da SPA, áreas do painel e comportamento responsivo
└── tasks.md             # Phase 2 (/speckit-tasks) — desatualizado, regenerar
```

### Source Code (repository root)

```text
backend/
├── app/
│   ├── main.py                 # cria o FastAPI e registra os routers
│   ├── config.py               # settings (pydantic-settings)
│   ├── db.py                   # engine/sessão SQLAlchemy
│   ├── celery_app.py           # app Celery: broker RabbitMQ, filas, DLQ, schedule do Beat
│   ├── outbox_relay.py         # relay: outbox_message → RabbitMQ (SKIP LOCKED, confirms)
│   ├── seed.py
│   ├── shared/                 # erros, paginação, Clock, ObjectStorage (S3), Mailer (SMTP),
│   │                           # outbox.py (Outbox.enqueue, TaskPublisher),
│   │                           # images.py (validação/re-encode: portfólio e pedidos)
│   └── modules/
│       ├── accounts/           # conta (pessoa), sessões, verificação de e-mail,
│       │                       # recuperação de senha, exclusão de conta, `Me` com áreas
│       ├── artists/            # perfil de tatuador e portfólio
│       ├── studios/            # estúdio, dono, equipe (membros e convites), registro
│       │                       # de ações (AuditTrail), autorização `studio_access`,
│       │                       # exclusão de estúdio
│       ├── affiliations/       # convites e vínculos, dias de atendimento, expiração (job)
│       ├── scheduling/         # padrão semanal, exceções, avulsos, janela, disponibilidade,
│       │                       # agendamentos, remarcação, imagens de referência,
│       │                       # jobs (expirar, concluir, lembretes, resumo, anonimizar)
│       ├── discovery/          # busca
│       └── notifications/      # notificações pessoais e do estúdio + outbox de e-mail
│           # cada módulo: models.py (SQLAlchemy), domain/ (regras puras),
│           # repository.py, service.py (casos de uso), schemas.py (Pydantic),
│           # router.py (HTTP); opcionais: dependencies.py (injeção/autorização),
│           # tasks.py (tasks Celery finas que chamam os casos de uso)
├── migrations/                 # Alembic
├── tests/
│   ├── unit/                   # regras de domínio (disponibilidade, transições, janela,
│   │                           # dias, níveis de acesso)
│   └── integration/            # API + Postgres real (concorrência, autorização, jobs)
└── pyproject.toml

frontend/
├── src/
│   ├── app/                    # router, providers, layout responsivo, seletor de área
│   ├── api/                    # cliente HTTP + tipos gerados do OpenAPI
│   ├── features/
│   │   ├── auth/  account/  booking/  artist/  portfolio/
│   │   ├── studio/ (agenda, horarios, tatuadores, equipe, registro, perfil)
│   │   ├── discovery/  notifications/
│   └── components/             # UI compartilhada (botões, modal/bottom sheet, grade)
├── e2e/                        # Playwright (mobile + desktop)
└── package.json

infra/
└── nginx/nginx.conf            # / → SPA, /api → backend, /media → MinIO (só bucket público)

compose.yaml                    # nginx, backend, outbox-relay, celery-worker, celery-beat,
                                #   rabbitmq, postgres, minio (buckets portfolio público +
                                #   booking-refs privado), mailpit
compose.test.yaml               # postgres + rabbitmq + backend-tests
.env.example
.pre-commit-config.yaml
.github/workflows/ci.yml
```

**Structure Decision**: aplicação web com `backend/` (API FastAPI, relay de outbox e workers
Celery na mesma base de código e imagem)
e `frontend/` (React SPA), mais `infra/` para o nginx. O backend é organizado por módulo de
domínio (vertical slices). Com o modelo "conta = pessoa", estúdio e tatuador viraram módulos
próprios (`studios/`, `artists/`) em vez de perfis dentro de `accounts/`, o que mantém cada
módulo com uma única razão para mudar.

## Complexity Tracking

Sem violações da constituição a justificar.
