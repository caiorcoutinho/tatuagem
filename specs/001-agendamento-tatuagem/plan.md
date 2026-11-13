# Implementation Plan: Agendamento para Tatuadores e Estúdios

**Branch**: `001-agendamento-tatuagem` | **Date**: 2026-11-13 (revisão 5; revisão 4: duração definida
na aprovação, e-mail via Gmail, falta do cliente) | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-agendamento-tatuagem/spec.md`

## Summary

Aplicação web responsiva (mobile e desktop) para agendamento de sessões de tatuagem. Cada
**conta é uma pessoa**: pode ter perfil de tatuador, ser dona de um estúdio, ser membro da
equipe de vários estúdios e agendar como cliente; visitantes também agendam sem conta. O
estúdio é gerido pelo dono e pela equipe (membros), com registro de todas as ações de
gestão. Ele informa as macas, publica **faixas de atendimento** (padrão semanal, exceções e
avulsas), define os dias em que cada tatuador vinculado atende e a janela de antecedência.
A soma de sessões confirmadas simultâneas nunca passa das macas.

O cliente (visitante ou conta) envia um **pedido sem reserva** com data e faixa preferidas,
descrição, local, tamanho e até 3 imagens privadas; o **tatuador aprova definindo início e
duração** (múltiplos de 30 min) dentro das faixas da data, e só aí as regras de macas,
capacidade e sobreposição são aplicadas. Pedidos expiram em 48h (ou no início da faixa pedida),
podem ser remarcados pelo cliente e cancelados; sessões confirmadas podem ser canceladas até
o início e marcadas como "não compareceu" por até 7 dias. Cada cliente tem no máximo 2
pedidos pendentes. O visitante recebe o link de acesso na tela de confirmação e pode pedir o
reenvio. Há lembrete ao cliente 24h antes e resumo diário ao tatuador. Convites de vínculo expiram em
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
no RabbitMQ; workers Celery enviam os e-mails pela **API do Gmail** (conta @gmail.com no MVP;
Mailpit por SMTP em dev), com retentativa, tratamento de cota e dead-letter; o Celery Beat
dispara as periódicas (expirar pedidos e convites, concluir sessões, lembretes, resumo
diário, varredura de devoluções, anonimização, limpeza do registro). Detalhes e alternativas em [research.md](./research.md).

## Technical Context

**Language/Version**: Python 3.12 (backend); TypeScript 5 + React 19 (frontend)

**Primary Dependencies**: FastAPI, python-multipart, Pydantic v2, SQLAlchemy 2.0 (async) +
asyncpg, Alembic, argon2-cffi, boto3 (MinIO), Pillow, Celery 5 (broker RabbitMQ via
kombu/py-amqp), google-auth + httpx (API do Gmail), aiosmtplib (só dev/teste, Mailpit) ·
Vite, React Router,
TanStack Query, React Hook Form + Zod + @hookform/resolvers, Tailwind CSS,
openapi-typescript · @axe-core/playwright (testes). Justificativas em research.md.

**Storage**: PostgreSQL 16 (`citext`, `btree_gist`, `pg_trgm`; também a tabela de outbox);
MinIO (bucket público `portfolio`, bucket privado `booking-refs`); RabbitMQ 3.13 (filas
duráveis `email`, `default`, `email.dlq`)

**Testing**: pytest + pytest-asyncio + httpx (integração com Postgres real, relógio
controlado para jobs); Vitest + Testing Library; Playwright (e2e em 390×844 e 1440×900) + axe

**Target Platform**: navegadores modernos, mobile e desktop; servidor Linux com Docker Compose
(nginx, backend, outbox-relay, celery-worker, celery-beat, rabbitmq, postgres, minio;
mailpit em dev); envio de e-mail pela API do Gmail (HTTPS) em produção

**Project Type**: aplicação web (frontend SPA + backend API + relay de outbox + workers
Celery)

**Performance Goals**: portfólio com 100 trabalhos visível em < 2 s (SC-004); envio de pedido
concluído em < 3 min (SC-001)

**Constraints**: 0% de sobreposição de sessões confirmadas, 0% de excesso de macas e de
capacidade de faixa (checados na aprovação) e no máximo 2 pendentes por contato sob
concorrência (SC-003, FR-007a, FR-007c, FR-011c); e-mail limitado a ~500 destinatários/dia
(conta Gmail pessoal, risco aceito na spec); toda ação de gestão do
estúdio auditada na mesma transação (FR-023); remoção de membro com efeito imediato
(FR-021); upload ≤ 10 MB por imagem; faixas e datas no fuso do estúdio; UI utilizável a partir de
360 px; LGPD: contato, detalhes e imagens restritos às partes e anonimizados após 12 meses

**Scale/Scope**: MVP para até ~3 mil agendamentos/mês (~80–150/dia), teto imposto pela cota
de ~500 e-mails/dia da conta Gmail pessoal com ~6 e-mails por sessão confirmada (spec,
Assumptions); dezenas de estúdios e algumas centenas de tatuadores; cerca de 30 telas;
servidor único. Gatilho de migração do e-mail: `mail_sent_today` acima de 80% da cota de
forma recorrente (alerta da T145)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio | Verificação | Status |
|---|---|---|
| I. Qualidade de Código | ruff + mypy (backend), ESLint + Prettier (frontend), testes obrigatórios de regra de negócio, CI bloqueando merge (R11); processamento de imagem e validação de agendamento compartilhados, sem duplicação | ✅ |
| II. Código Enxuto | Toda dependência externa tem motivo em research.md; Celery + RabbitMQ justificados em R6 (retentativa com backoff, dead-letter, escala de consumidores e periódicas, por decisão do usuário); sem motor de busca, lib de masonry ou framework de permissões | ✅ |
| III. SOLID | Módulos de domínio; regras puras em `domain/`; autorização isolada em dependências (`studio_access`, `require_artist`, `booking_party`) e não espalhada pelos routers; auditoria num serviço único (`AuditTrail`); `Protocol`s para relógio, storage, providers entre módulos e `Mailer` (Gmail em produção, SMTP/Mailpit em dev: troca sem tocar nos casos de uso) | ✅ |
| IV. Clean Code | Nomes de domínio da spec (`booking`, `affiliation`, `studio_member`, `seats`, `weekdays`); erros com códigos explícitos; no máximo 3 parâmetros, com objetos de parâmetros (`ScheduleSnapshot`, `DateRange`, `NotificationRequest`, `AuditEntry`) | ✅ |
| V. YAGNI | Só dois níveis de acesso (dono, membro) sem RBAC genérico; faixas calculadas; pedido sem reserva (não há lock nem ocupação na criação); cliente Gmail mínimo (`google-auth` + `httpx`) em vez do SDK completo; um único mecanismo assíncrono (outbox → RabbitMQ → Celery) para e-mails e periódicas; `Protocol` só com duas implementações; sem backup automatizado (fora do escopo pela spec) | ✅ |
| VI. Commits Semânticos | `conventional-pre-commit` no hook `commit-msg` e checagem no CI | ✅ |
| Padrões de Qualidade | pre-commit local + CI; `.env` fora do git com `.env.example` | ✅ |

**Gate (pré-pesquisa)**: aprovado, sem violações.

**Re-check pós-design (revisão 5, 2026-11-13)**: aprovado, sem mudança de arquitetura nem
dependência nova. Expiração no início da faixa pedida (R15), escala do MVP alinhada à cota de
e-mail e correções de coerência F3 (R14), C1 (check do data-model), F4 (R16), F6 (termo
`window`) e F7 (submódulo do portfólio) do último `/speckit-analyze`.

**Re-check pós-design (revisão 4, 2026-11-13)**: aprovado. Mover as regras de ocupação para a
aprovação (R4) **simplifica** a criação do pedido (só locks do cliente) e concentra a
concorrência num único ponto. Dependência nova: `google-auth` (renovação do token OAuth do
Gmail, R23); `httpx` passa de dependência de teste a de produção. Os achados D1
(Complexity Tracking) e F1 (e-mail `booking_received`) do último `/speckit-analyze` foram
corrigidos nesta revisão.

**Re-check pós-design (revisão 3, 2026-11-13)**: aprovado, sem dependência nova. A troca de
"perfil único por conta" por "conta = pessoa" (research.md R19) **simplifica** a
autorização: um único mecanismo por recurso substitui papéis fixos e já cobre dono, membro,
tatuador e cliente. Equipe e registro de ações (R20), dias de atendimento e janela (R21) e
lembretes (R22) reaproveitam transação, outbox e tasks Celery. A troca do worker caseiro
por Celery + RabbitMQ (R6, decisão do usuário) mantém a atomicidade pelo outbox
transacional e reaproveita os casos de uso assíncronos nas tasks (`asyncio.run`), sem
camada síncrona duplicada. Revisões anteriores corrigiram os achados D1, D2 e F3 do
`/speckit-analyze`.

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
│   ├── tools/gmail_authorize.py # gera o refresh token OAuth da conta Gmail (uso único)
│   ├── shared/                 # erros, paginação, Clock, ObjectStorage (S3),
│   │                           # mailer.py (Mailer: GmailApiMailer | SmtpMailer),
│   │                           # outbox.py (Outbox.enqueue, TaskPublisher),
│   │                           # images.py (validação/re-encode: portfólio e pedidos)
│   └── modules/
│       ├── accounts/           # conta (pessoa), sessões, verificação de e-mail,
│       │                       # recuperação de senha, exclusão de conta, `Me` com áreas
│       ├── artists/            # perfil de tatuador
│       │   └── portfolio/      # submódulo do portfólio (models, repository, service, router)
│       ├── studios/            # estúdio, dono, equipe (membros e convites), registro
│       │                       # de ações (AuditTrail), autorização `studio_access`,
│       │                       # exclusão de estúdio
│       ├── affiliations/       # convites e vínculos, dias de atendimento, expiração (job)
│       ├── scheduling/         # padrão semanal, exceções, avulsos, janela, disponibilidade,
│       │                       # pedidos, aprovação com intervalo livre, remarcação, falta,
│       │                       # imagens de referência, link do visitante,
│       │                       # jobs (expirar, concluir, lembretes, resumo, anonimizar)
│       ├── discovery/          # busca
│       └── notifications/      # notificações pessoais e do estúdio, envio via outbox,
│                               # varredura de devoluções do Gmail
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

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Mensageria com 4 processos e um broker (API + `outbox-relay` + `celery-worker` + `celery-beat` + RabbitMQ) em vez de um único worker lendo o Postgres | Decisão do usuário (2026-11-13). Traz retentativa com backoff, dead-letter, tratamento de cota do Gmail, escala independente de consumidores e agendamento de periódicas sem infraestrutura escrita à mão (research.md R6) | O worker único com loop `asyncio` (plano anterior) era suficiente para o volume da v1, mas exigiria implementar à mão backoff, DLQ, idempotência de reentrega e agendamento; foi substituído por escolha explícita do usuário. A atomicidade é preservada pelo outbox transacional |
| Duas implementações de `Mailer` (Gmail API e SMTP) | Produção usa a API do Gmail (decisão do usuário); desenvolvimento e testes precisam funcionar sem conta Google (Mailpit) | Usar só o Gmail impediria rodar e testar localmente sem credenciais reais; usar só SMTP não permite ler devoluções da caixa do Gmail (FR-002b) |
