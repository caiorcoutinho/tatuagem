# Implementation Plan: Agendamento para Tatuadores e Estúdios

**Branch**: `001-agendamento-tatuagem` | **Date**: 2026-11-12 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-agendamento-tatuagem/spec.md`

## Summary

Aplicação web responsiva (mobile e desktop) para agendamento de sessões de tatuagem. Há
três perfis: Estúdio, Tatuador e Cliente (que também pode agendar como visitante). O estúdio
publica horários por um padrão semanal com exceções; cada horário tem uma capacidade de
vagas. Todo agendamento exige estúdio + tatuador com vínculo ativo, nasce pendente, é
aprovado pelo tatuador e expira em 48h sem resposta. O tatuador publica um portfólio em
grade de imagens.

Abordagem técnica: SPA **React** + API **FastAPI**, **PostgreSQL** (regras de concorrência
garantidas no banco por constraint de exclusão e advisory lock), **MinIO** para imagens do
portfólio, **nginx** na frente servindo o frontend, a API e as imagens no mesmo domínio, e um
**worker** que expira pendentes e envia e-mails pelo padrão outbox. Detalhes e alternativas
em [research.md](./research.md).

## Technical Context

**Language/Version**: Python 3.12 (backend); TypeScript 5 + React 19 (frontend)

**Primary Dependencies**: FastAPI, Pydantic v2, SQLAlchemy 2.0 (async) + asyncpg, Alembic,
argon2-cffi, boto3 (MinIO), Pillow, aiosmtplib · Vite, React Router, TanStack Query, React
Hook Form + Zod, Tailwind CSS, openapi-typescript

**Storage**: PostgreSQL 16 (`btree_gist`, `pg_trgm`); MinIO (bucket `portfolio`, imagens)

**Testing**: pytest + pytest-asyncio + httpx (integração com Postgres real); Vitest +
Testing Library; Playwright (e2e em 390×844 e 1440×900)

**Target Platform**: navegadores modernos, mobile e desktop; servidor Linux com Docker Compose
(nginx, backend, worker, postgres, minio; mailpit em dev)

**Project Type**: aplicação web (frontend SPA + backend API + worker)

**Performance Goals**: portfólio com 100 trabalhos visível em < 2 s (SC-004); fluxo de
agendamento concluído em < 3 min (SC-001); API p95 < 300 ms nas rotas de leitura

**Constraints**: 0% de sobreposição de agendamentos sob concorrência (SC-003); upload ≤ 10 MB;
horários no fuso do estúdio; UI utilizável a partir de 360 px de largura

**Scale/Scope**: v1 para centenas de estúdios, alguns milhares de tatuadores e dezenas de
milhares de agendamentos/mês; cerca de 20 telas; servidor único

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio | Verificação | Status |
|---|---|---|
| I. Qualidade de Código | ruff + mypy (backend), ESLint + Prettier (frontend), testes obrigatórios de regra de negócio, CI bloqueando merge (R11) | ✅ |
| II. Código Enxuto | Cada dependência tem motivo registrado em research.md; sem fila externa, sem motor de busca, sem lib de masonry | ✅ |
| III. SOLID | Backend por módulos de domínio; regras puras em `domain/` sem dependência de infraestrutura; serviços dependem de `Protocol`s de repositório e storage (DIP); routers só traduzem HTTP | ✅ |
| IV. Clean Code | Nomes de domínio da spec (`booking`, `affiliation`, `weekly_rule`); erros tratados com códigos explícitos (contracts/api.md); formatter automatizado | ✅ |
| V. YAGNI | Ocorrências de horário calculadas em vez de materializadas; sem pagamentos, chat ou reviews; `completed` automático, sem fluxo manual; abstrações só onde há duas implementações (repositório real + fake de teste) | ✅ |
| VI. Commits Semânticos | `conventional-pre-commit` no hook `commit-msg` e checagem no CI | ✅ |
| Padrões de Qualidade | pre-commit local + CI; `.env` fora do git com `.env.example` | ✅ |

**Gate (pré-pesquisa)**: aprovado, sem violações.
**Re-check pós-design (Phase 1)**: aprovado. O desenho manteve um backend único com worker
no mesmo código (não é um projeto separado). Constraint de exclusão e advisory lock são SQL
nativo, sem dependência nova. Nenhuma entrada em Complexity Tracking é necessária.

## Project Structure

### Documentation (this feature)

```text
specs/001-agendamento-tatuagem/
├── plan.md              # Este arquivo
├── research.md          # Phase 0: decisões técnicas
├── data-model.md        # Phase 1: entidades, regras e transições
├── quickstart.md        # Phase 1: guia de validação ponta a ponta
├── contracts/
│   ├── api.md           # Contrato REST /api/v1
│   └── ui-routes.md     # Rotas da SPA e comportamento responsivo
└── tasks.md             # Phase 2 (/speckit-tasks — ainda não criado)
```

### Source Code (repository root)

```text
backend/
├── app/
│   ├── main.py                 # cria o FastAPI e registra os routers
│   ├── config.py               # settings (pydantic-settings)
│   ├── db.py                   # engine/sessão SQLAlchemy
│   ├── worker.py               # loop: expira pendentes, conclui sessões, envia e-mails
│   ├── seed.py
│   ├── shared/                 # erros de domínio, paginação, relógio (Clock), storage (S3)
│   └── modules/
│       ├── accounts/           # usuários, sessões, perfis, verificação de e-mail
│       ├── affiliations/       # convites e vínculos
│       ├── scheduling/         # padrão semanal, exceções, disponibilidade, agendamentos
│       ├── portfolio/          # trabalhos e processamento de imagem
│       ├── discovery/          # busca
│       └── notifications/      # notificações in-app + outbox de e-mail
│           # cada módulo: domain/ (regras puras), service.py (casos de uso),
│           # repository.py (SQLAlchemy), router.py (HTTP), schemas.py (Pydantic)
├── migrations/                 # Alembic
├── tests/
│   ├── unit/                   # regras de domínio (disponibilidade, transições, expiração)
│   └── integration/            # API + Postgres real (concorrência, constraints)
└── pyproject.toml

frontend/
├── src/
│   ├── app/                    # router, providers, layout responsivo
│   ├── api/                    # cliente HTTP + tipos gerados do OpenAPI
│   ├── features/
│   │   ├── auth/  booking/  schedule/  affiliations/
│   │   ├── portfolio/  discovery/  dashboard/  notifications/
│   └── components/             # UI compartilhada (botões, modal/bottom sheet, grade)
├── e2e/                        # Playwright (mobile + desktop)
└── package.json

infra/
└── nginx/nginx.conf            # / → SPA, /api → backend, /media → MinIO

compose.yaml                    # nginx, backend, worker, postgres, minio, mailpit
compose.test.yaml               # postgres + backend-tests
.env.example
.pre-commit-config.yaml
.github/workflows/ci.yml
```

**Structure Decision**: aplicação web com `backend/` (FastAPI + worker, mesma base de código)
e `frontend/` (React SPA), mais `infra/` para o nginx. O backend é organizado por módulo de
domínio (vertical slices), alinhado às entidades da spec, para que cada história de usuário
mexa em poucos módulos.

## Complexity Tracking

Sem violações da constituição a justificar.
