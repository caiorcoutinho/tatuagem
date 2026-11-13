---
description: "Task list for 001-agendamento-tatuagem (revisão 5: pedido sem reserva, duração na aprovação, Gmail, falta, expiração no início da faixa)"
---

# Tasks: Agendamento para Tatuadores e Estúdios

**Input**: Design documents from `/specs/001-agendamento-tatuagem/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/api.md](./contracts/api.md),
[contracts/ui-routes.md](./contracts/ui-routes.md), [quickstart.md](./quickstart.md)

**Tests**: a constituição exige teste automatizado para toda lógica de negócio (Padrões de
Qualidade). Cada fase inclui testes de domínio (unit) e de API (integração com Postgres
real). Escreva o teste antes da implementação e confira que ele falha.

**Organization**: fases por história de usuário, em ordem de prioridade: P1 (US1, US2,
US7), P2 (US3, US4, US6), P3 (US5).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: pode rodar em paralelo (arquivos diferentes, sem dependência pendente)
- **[Story]**: história da spec (US1–US7)

## Path Conventions

- Backend: `backend/app/`, testes em `backend/tests/`, migrações em `backend/migrations/versions/`
- Módulos: `backend/app/modules/<módulo>/` com `models.py`, `domain/` (regras puras, sem
  SQLAlchemy/FastAPI/Celery), `repository.py`, `service.py`, `schemas.py`, `router.py` e,
  quando preciso, `dependencies.py` e `tasks.py` (tasks Celery finas que chamam o serviço)
- Frontend: `frontend/src/`, e2e em `frontend/e2e/`
- Erros seguem o formato e os códigos de [contracts/api.md](./contracts/api.md)
- Assinaturas com no máximo 3 parâmetros (constituição IV): use objetos de parâmetros
- Toda ação de gestão de estúdio grava `AuditTrail.record` na mesma transação (FR-023)
- Todo e-mail sai por `NotificationService.notify` → `Outbox.enqueue` na mesma transação;
  nunca chamar Celery nem `Mailer` direto de um caso de uso

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: repositório, ferramentas e ambiente Docker funcionando vazios

- [ ] T001 Create root layout per plan.md: `backend/`, `frontend/`, `infra/nginx/`, root `.gitignore` (`.env`, `node_modules/`, `__pycache__/`, `.venv/`) and `.env.example` with all variables (Postgres, MinIO buckets `portfolio`/`booking-refs`, `RABBITMQ_URL`, `MAIL_BACKEND=smtp|gmail`, SMTP for Mailpit, `GMAIL_SENDER`, `GMAIL_CLIENT_ID`, `GMAIL_CLIENT_SECRET`, `GMAIL_REFRESH_TOKEN`, `APP_ORIGIN`, `SESSION_TTL_DAYS=30`)
- [ ] T002 Initialize backend in `backend/pyproject.toml` (Python 3.12; deps: fastapi, uvicorn[standard], python-multipart, pydantic-settings, sqlalchemy[asyncio], asyncpg, alembic, argon2-cffi, boto3, pillow, celery, google-auth, httpx, aiosmtplib; dev: pytest, pytest-asyncio, ruff, mypy) with ruff config and `mypy --strict` for `app/modules/*/domain`
- [ ] T003 [P] Initialize frontend in `frontend/` with Vite + React 19 + TypeScript; deps: react-router, @tanstack/react-query, react-hook-form, zod, @hookform/resolvers, tailwindcss; dev: vitest, @testing-library/react, @playwright/test, @axe-core/playwright, openapi-typescript, eslint, prettier (`frontend/package.json`, `vite.config.ts`, `tailwind.config.ts`, `eslint.config.js`)
- [ ] T004 [P] Create `backend/Dockerfile` (single image; commands: `uvicorn app.main:app`, `python -m app.outbox_relay`, `celery -A app.celery_app worker -Q email,default`, `celery -A app.celery_app beat`) and `frontend/Dockerfile` (multi-stage build producing static files)
- [ ] T005 [P] Configure nginx in `infra/nginx/nginx.conf`: `/` → SPA with `try_files ... /index.html`; `/api/` → `backend:8000`; `/media/` → `minio:9000/portfolio/` only (GET, long immutable cache); `client_max_body_size 32m`; gzip; security headers
- [ ] T006 Create `compose.yaml`: nginx (8080), backend, outbox-relay, celery-worker, celery-beat (1 réplica), rabbitmq:3.13-management (15672), postgres:16, minio (9001), mailpit (8025), one-shot `minio-init` creating `portfolio` (anonymous read) and `booking-refs` (private); create `compose.test.yaml` with postgres + rabbitmq + mailpit + `backend-tests`
- [ ] T007 [P] Configure `.pre-commit-config.yaml`: ruff (lint + format), eslint, prettier, `conventional-pre-commit` on `commit-msg`
- [ ] T008 [P] Create CI in `.github/workflows/ci.yml`: ruff, mypy, pytest via `compose.test.yaml`, frontend lint + vitest, Playwright e2e, conventional commit check

**Checkpoint**: `docker compose up` sobe todos os serviços; hooks e CI executam

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: núcleo, mensageria e e-mail, contas e autenticação, perfil de tatuador, estúdio
com níveis de acesso e auditoria, shell do frontend

**⚠️ CRITICAL**: nenhuma história começa antes desta fase terminar

### Núcleo do backend

- [ ] T009 Implement settings in `backend/app/config.py` (pydantic-settings) and async engine/session in `backend/app/db.py` (`make_engine(settings)` with `NullPool` option for Celery processes)
- [ ] T010 Initialize Alembic in `backend/migrations/` (async env; `target_metadata` from `backend/app/models_registry.py` importing every module's models)
- [ ] T011 [P] Implement domain errors in `backend/app/shared/errors.py` (`DomainError(code, message)`, `NotFound`, `Forbidden`, `Conflict`, `ValidationFailed`) and FastAPI handlers rendering `{"error": {code, message, fields?}}`
- [ ] T012 [P] Implement cursor pagination in `backend/app/shared/pagination.py` (`PageRequest(cursor, limit 1..50)`, `Page{items, next_cursor}`)
- [ ] T013 [P] Implement `Clock` protocol + `SystemClock` in `backend/app/shared/clock.py`
- [ ] T014 [P] Implement `ObjectStorage` protocol + `S3ObjectStorage` (`put(StoredObject)`, `get_stream(ObjectRef)`, `delete_prefix(ObjectRef)`, `public_url(key)` → `/media/{key}`) in `backend/app/shared/storage.py`
- [ ] T015 [P] Write unit tests in `backend/tests/unit/shared/test_images.py` (fixtures in `backend/tests/fixtures/images/`): accepts JPEG/PNG/WebP, rejects renamed text file and > 10 MB, strips EXIF, produces WebP variants with requested max widths and returns dimensions
- [ ] T016 [P] Implement `process_image(upload: ImageUpload, spec: VariantSpec) -> list[ImageVariant]` in `backend/app/shared/images.py` (Pillow) to pass T015

### Mensageria e e-mail (research.md R6, R23)

- [ ] T017 [P] Write unit tests in `backend/tests/unit/shared/test_mailer.py`: MIME building (UTF-8 subject/body, `Message-ID` generated and returned), `GmailApiMailer` maps HTTP 429 / `rateLimitExceeded` / `dailyLimitExceeded` to `MailQuotaExceeded`, `invalid_grant` to `MailAuthError`, other 5xx to `MailTransientError` (httpx mocked transport)
- [ ] T018 Implement `Mailer` protocol (`send(message: OutgoingEmail) -> SentEmail`) with `GmailApiMailer` (google-auth refresh token → access token; `POST gmail/v1/users/me/messages/send` via httpx) and `SmtpMailer` (aiosmtplib → Mailpit), selected by `MAIL_BACKEND`, in `backend/app/shared/mailer.py` to pass T017
- [ ] T019 [P] Implement one-time OAuth helper `backend/app/tools/gmail_authorize.py` (local loopback consent with scopes `gmail.send` + `gmail.modify`, prints refresh token; README note that the OAuth app must be "In production")
- [ ] T020 Create `Notification` (incl. `studio_id`, `payload`, `email_message_id`, `email_failed_at`, `delivery_failure_reason`), `OutboxMessage`, `JobRun` models in `backend/app/modules/notifications/models.py` and migration `backend/migrations/versions/0001_notifications_outbox.py`
- [ ] T021 Implement `Outbox.enqueue(session, message: OutboxRequest)` and `TaskPublisher` protocol + `CeleryTaskPublisher` (`send_task`, `task_id` = outbox id, confirm publish) in `backend/app/shared/outbox.py`
- [ ] T022 Implement `NotificationService.notify(session, request: NotificationRequest)` in `backend/app/modules/notifications/service.py` (personal or studio recipient; studio e-mail goes to owner per FR-024; enqueues `notifications.send_email`) and pt-BR templates for every type in data-model.md (incl. `booking_received` with guest link, `guest_links`) in `backend/app/modules/notifications/templates.py`
- [ ] T023 Create Celery app in `backend/app/celery_app.py`: RabbitMQ broker, durable queues `email`/`default`, `email.dlq` via dead-letter exchange, `acks_late`, `task_reject_on_worker_lost`, prefetch 1 on `email`, Beat schedule registry filled by modules, timezone `America/Sao_Paulo`; helper `run_async(coro)` with `NullPool` engine
- [ ] T024 Implement outbox relay in `backend/app/outbox_relay.py`: every 1 s, `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 100` on unpublished rows, publish via `TaskPublisher`, set `published_at` in same transaction, rollback and increment `publish_attempts` on broker failure; graceful shutdown
- [ ] T025 Implement task `notifications.send_email` in `backend/app/modules/notifications/tasks.py`: reload notification `FOR UPDATE SKIP LOCKED`, skip if `email_sent_at` set, send via `Mailer`, store `email_message_id` and `email_sent_at`; `MailQuotaExceeded` → `retry(countdown up to 1h)` without consuming failure attempts; `MailAuthError` → log error, keep message for later (no tight loop); transient errors → exponential backoff + jitter, max 6, then `email_failed_at` and reject to `email.dlq`; increment a daily sent counter
- [ ] T026 Implement notification routes in `backend/app/modules/notifications/router.py`: `GET /me/notifications`, `POST /me/notifications/read` (studio routes come in T046)
- [ ] T027 [P] Write integration tests in `backend/tests/integration/test_messaging.py`: notify creates notification + outbox row atomically (rollback leaves neither), relay publishes via `FakeTaskPublisher` and marks `published_at`, publish failure keeps row pending, two relays never publish the same row, `send_email` idempotent on redelivery, quota error reschedules without counting attempts, auth error does not loop, failure path sets `email_failed_at`
- [ ] T028 Create FastAPI app in `backend/app/main.py`: `/api/v1` prefix, error handlers, Origin check middleware for mutating methods (research.md R3), router registration, `GET /api/v1/health` (db, storage, broker, `mail` ok/error, `mail_sent_today`, `outbox_backlog`)

### Contas e autenticação

- [ ] T029 Create models `User` (sem papel; `phone_norm`), `Session`, `PasswordResetToken`, `EmailVerificationToken` in `backend/app/modules/accounts/models.py` and migration `backend/migrations/versions/0002_accounts.py` (extensions `citext`, `btree_gist`, `pg_trgm`)
- [ ] T030 [P] Write unit tests in `backend/tests/unit/accounts/test_rules.py`: slug generation with uniqueness suffix, city normalization, phone normalization (`(11) 98888-7777` and `5511988887777` equal), email normalization, password ≥ 8, IANA timezone validation
- [ ] T031 [P] Implement pure rules in `backend/app/modules/accounts/domain/rules.py` (`make_slug`, `normalize_city`, `normalize_phone`, `normalize_email`, `validate_password`, `validate_timezone`) to pass T030
- [ ] T032 Implement `backend/app/modules/accounts/security.py`: Argon2 hashing; session token (32 bytes, SHA-256 stored, cookie `session` HttpOnly/Secure/SameSite=Lax, 30-day sliding); generic one-time token helper (reset 1h, verification 48h)
- [ ] T033 Implement `AccountRepository` + `AccountService` in `backend/app/modules/accounts/repository.py` and `service.py`: register (verification e-mail via `NotificationService`), login, logout, update `name`/`phone`, verify email (then calls `GuestBookingLinker` protocol, no-op until US2), resend verification, password reset request (always silent) and confirm (deletes all sessions); `email_taken`, `invalid_token`
- [ ] T034 Implement auth dependencies in `backend/app/modules/accounts/dependencies.py`: `current_user`, `optional_user`
- [ ] T035 Implement routes in `backend/app/modules/accounts/router.py` and `schemas.py`: `/auth/register`, `/auth/login`, `/auth/logout`, `GET /auth/me` (`Me` with `areas` from an `AreasProvider` protocol filled by artists/studios modules), `PATCH /me`, `/auth/verify-email`, `/auth/verify-email/resend`, `/auth/password-reset/request`, `/auth/password-reset/confirm`
- [ ] T036 Create pytest fixtures in `backend/tests/conftest.py`: migrated DB with per-test rollback, `FakeClock`, `FakeStorage`, `FakeMailer`, `FakeTaskPublisher`, httpx `client`, factories `make_account(verified=True)`, `make_artist`, `make_studio(owner)`, `add_member(studio, account)` returning logged-in clients
- [ ] T037 [P] Write integration tests in `backend/tests/integration/test_auth.py`: register/login/logout, duplicate email → 409 `email_taken`, `/auth/me` areas empty for new account, verification flow and resend, password reset silent for unknown email, reset invalidates all sessions and reused token → 400 `invalid_token`, cross-origin POST rejected

### Perfil de tatuador

- [ ] T038 Create `ArtistProfile` (unique `user_id`, `display_name`, `deleted_at`), `Style`, `ArtistStyle` in `backend/app/modules/artists/models.py` and migration `backend/migrations/versions/0003_artists.py`
- [ ] T039 Implement `ArtistRepository` + `ArtistService` + routes in `backend/app/modules/artists/repository.py`, `service.py`, `router.py`: `POST /me/artist` (409 `artist_profile_exists`), `PATCH /me/artist`, `PUT /me/artist/photo` (`process_image` → 400px WebP in `portfolio`), `GET /styles`; `require_artist` in `backend/app/modules/artists/dependencies.py`; register artist area in `AreasProvider`
- [ ] T040 [P] Create seed script `backend/app/seed.py` with fixed style list (fineline, old-school, realismo, blackwork, aquarela, tribal, geométrico, oriental, lettering, pontilhismo) and `--demo` personas from quickstart.md (Ana, Bruno, Carla, Davi)

### Estúdio: acesso e auditoria

- [ ] T041 Create `Studio` (unique partial `owner_user_id`, `seats`, `min_notice_hours`, `max_advance_days`, `deleted_at`), `StudioMember`, `StudioMemberInvite`, `StudioAuditLog` in `backend/app/modules/studios/models.py` and migration `backend/migrations/versions/0004_studios.py`
- [ ] T042 [P] Write unit tests in `backend/tests/unit/studios/test_access.py`: access level resolution (owner / member / none) and the permission table in data-model.md
- [ ] T043 [P] Implement `resolve_access(studio, user_id) -> AccessLevel` and `can(level, StudioAction) -> bool` in `backend/app/modules/studios/domain/access.py` to pass T042
- [ ] T044 Implement `studio_access(level)` dependency in `backend/app/modules/studios/dependencies.py` (loads studio by `{studio_id}`, 404 if deleted, 403 if not owner/member, checked every request) and `AuditTrail.record(session, entry: AuditEntry)` in `backend/app/modules/studios/audit.py`
- [ ] T045 Implement publishability filters in `backend/app/shared/publishable.py` (`publishable_artists()` / `publishable_studios()` per data-model.md) used by every public query
- [ ] T046 Implement studio notification routes `GET /studios/{id}/notifications` and `POST /studios/{id}/notifications/read` (`studio_access(member)`, shared read state) in `backend/app/modules/notifications/router.py`
- [ ] T047 [P] Write integration tests in `backend/tests/integration/test_studio_access.py`: owner and member reach `member` routes, non-member → 403, member on owner route → 403, removed member loses access on next request without touching sessions, audit row written in same transaction (rollback leaves none), studio notifications visible to owner and members, e-mail only to owner

### Shell do frontend

- [ ] T048 Add `npm run gen:api` in `frontend/package.json` generating `frontend/src/api/schema.d.ts` from `/api/v1/openapi.json`; fetch wrapper `frontend/src/api/client.ts` (same-origin, JSON + multipart, typed `ApiError` with `code`)
- [ ] T049 Create app shell in `frontend/src/app/`: `router.tsx` (all routes from contracts/ui-routes.md, lazy), `providers.tsx`, `Layout.tsx` responsive (bottom nav < `lg`, sidebar ≥ `lg`), `AreaSwitcher.tsx` (areas from `/auth/me`, remembered in `localStorage`, dropdown on mobile), `useCurrentArea`, and 403 handling that drops a lost studio from the switcher
- [ ] T050 [P] Create shared UI in `frontend/src/components/`: `Button`, `TextField`, `Select`, `Modal` (bottom sheet on mobile), `Skeleton`, `EmptyState`, `ErrorMessage` (API code → pt-BR), `ConfirmWithPassword`, `CopyLinkField`, all with ≥ 44px targets
- [ ] T051 Implement auth pages in `frontend/src/features/auth/`: `LoginPage`, `RegisterPage` (name, email, phone, password), `VerifyEmailPage`, `ForgotPasswordPage` (neutral message), `ResetPasswordPage`, `useSession` hook and `RequireAuth` guard
- [ ] T052 [P] Implement `frontend/src/features/account/AccountPage.tsx` (`/painel/conta`: name, phone, verification status + resend) and `UnverifiedBanner.tsx` shown in the dashboard layout
- [ ] T053 [P] Implement `frontend/src/features/artist/ArtistProfilePage.tsx` (`/painel/tatuador/perfil`: create or edit display name, bio, city, styles, photo) and "Criar perfil de tatuador" entry in the empty dashboard
- [ ] T054 [P] Implement notifications in `frontend/src/features/notifications/` (personal and studio lists by current area, mark as read, unread badge, polling 60 s)
- [ ] T055 [P] Configure Playwright in `frontend/playwright.config.ts` (projects `mobile` 390×844 and `desktop` 1440×900) and helpers in `frontend/e2e/helpers.ts` (personas via API, Mailpit reader, e-mail confirmation)

**Checkpoint**: contas, verificação, recuperação de senha, perfil de tatuador, acesso a
estúdio, auditoria e e-mails (Mailpit) ponta a ponta; `pytest` verde

---

## Phase 3: User Story 1 - Estúdio configura agenda e vincula tatuadores (Priority: P1) 🎯 MVP

**Goal**: conta cria seu estúdio (macas, fuso), convida tatuadores (vínculo ativo após
aceite, convite expira em 24h), define dias de atendimento por tatuador, faixas (padrão
semanal, exceções, avulsas) e janela de antecedência; datas oferecidas consultáveis.

**Independent Test**: quickstart.md V1 (e V13 passos 1–2).

### Tests for User Story 1

- [ ] T056 [P] [US1] Write unit tests in `backend/tests/unit/scheduling/test_windows.py`: weekly rules in studio timezone → UTC (zone with and without DST), override blocked / windows, one-off added on its date, overlapping origins allowed, offered dates clipped by booking window (min notice 0/24h, max 7/60 days) and artist weekdays bitmask, date hidden when no window has 30 free minutes for the artist given confirmed-session load
- [ ] T057 [P] [US1] Write unit tests in `backend/tests/unit/scheduling/test_schedule_validation.py`: `start < end`, `1 ≤ capacity ≤ seats`, overlapping weekly rules on same weekday rejected, override windows non-overlapping, booking window ranges 0–72h / 7–180 days, weekdays ≥ 1
- [ ] T058 [P] [US1] Write unit tests in `backend/tests/unit/affiliations/test_transitions.py`: invited→active/declined/expired/cancelled, active→ended (studio or artist), accept/decline require `now < expires_at`, others raise `invitation_not_valid`/`invalid_transition`
- [ ] T059 [P] [US1] Write integration tests in `backend/tests/integration/test_studio_setup.py`: create studio (second → 409 `already_owns_studio`), owner edits profile and seats, member edits booking window but seats → 403, each change audited
- [ ] T060 [P] [US1] Write integration tests in `backend/tests/integration/test_affiliations.py`: invite creates notification + audit, duplicate → 409 `affiliation_exists`, accept sets weekdays = all, expired/cancelled accept → 409, cancel and end by studio, expiry task (FakeClock) sets `expired` and notifies studio, re-invite after terminal status, weekdays update audited, public studio page lists only publishable active artists
- [ ] T061 [P] [US1] Write integration tests in `backend/tests/integration/test_schedule.py`: rules replace, override block/windows, one-off CRUD, availability returns dates + windows, requires `artist` (400) and active affiliation (409), hides unpublishable owner/artist, non-member → 403, every mutation audited

### Implementation for User Story 1

- [ ] T062 [US1] Implement `StudioRepository` + `StudioService` in `backend/app/modules/studios/repository.py` and `service.py`: create (owner = current user), update profile (owner), update seats (owner; `seats_below_bookings` via `ConfirmedLoadProvider` protocol, no-op until US2), update booking window (member), photo; all mutations audited; register owned/member studios in `AreasProvider`
- [ ] T063 [US1] Implement studio routes in `backend/app/modules/studios/router.py`: `POST /studios`, `GET /studios/{slug}`, `GET /studios/{id}/settings`, `PATCH /studios/{id}`, `PATCH /studios/{id}/booking-window`, `PUT /studios/{id}/photo`
- [ ] T064 [P] [US1] Create `Affiliation` model (status enum, `expires_at`, `weekdays` smallint, partial unique index) in `backend/app/modules/affiliations/models.py`
- [ ] T065 [P] [US1] Create `WeeklyRule`, `DateOverride` (jsonb windows), `OneOffSlot` models in `backend/app/modules/scheduling/models.py`
- [ ] T066 [US1] Create migration `backend/migrations/versions/0005_affiliations_schedule.py` for T064–T065
- [ ] T067 [P] [US1] Implement affiliation state machine and weekdays bitmask helpers in `backend/app/modules/affiliations/domain/transitions.py` and `weekdays.py` to pass T058
- [ ] T068 [P] [US1] Implement pure windows logic in `backend/app/modules/scheduling/domain/windows.py` (`windows_for_date(schedule: ScheduleSnapshot, day)`, `offered_dates(schedule, query: OfferQuery)`, `free_intervals(windows, load: ConfirmedLoad)` with 30-min grid) and validators in `backend/app/modules/scheduling/domain/validation.py` to pass T056–T057
- [ ] T069 [US1] Implement `AffiliationRepository` + `AffiliationService` in `backend/app/modules/affiliations/repository.py` and `service.py`: invite by artist slug, cancel, end (studio side), accept/decline/end (artist side), set weekdays; notifications; audit for studio-side actions; `active_affiliation(studio_id, artist_id)` for other modules
- [ ] T070 [US1] Implement affiliation routes in `backend/app/modules/affiliations/router.py`: `/studios/{id}/affiliations` (POST, GET), `.../{aid}/cancel`, `.../{aid}/weekdays`, `.../{aid}/end`; `/me/affiliations` (GET), `.../{aid}/accept|decline|end`
- [ ] T071 [US1] Implement periodic task `affiliations.expire_invites` (1 min) in `backend/app/modules/affiliations/tasks.py` and register it in `backend/app/celery_app.py`
- [ ] T072 [US1] Implement `ScheduleRepository` + `ScheduleService` in `backend/app/modules/scheduling/repository.py` and `schedule_service.py`: replace weekly rules, upsert/delete override, one-off CRUD with `capacity ≤ seats`; `slot_has_bookings` via `ConfirmedLoadProvider` (no-op until US2); audit every mutation
- [ ] T073 [US1] Implement `AvailabilityService.offered_dates(query: OfferQuery)` in `backend/app/modules/scheduling/availability_service.py`: loads schedule + studio window + affiliation weekdays, requires publishable studio/artist and active affiliation, uses `ConfirmedLoadProvider` and `ArtistBusyProvider` (no-op until US2)
- [ ] T074 [US1] Implement routes in `backend/app/modules/scheduling/router.py`: `/studios/{id}/schedule/rules`, `/studios/{id}/schedule/overrides/{date}`, `/studios/{id}/schedule/one-off-slots[/{sid}]` (`studio_access(member)`), `GET /studios/{slug}/availability`
- [ ] T075 [P] [US1] Implement `frontend/src/features/studio/CreateStudioPage.tsx` (name, address, city, timezone default `America/Sao_Paulo`, seats) and `frontend/src/features/studio/StudioProfilePage.tsx` (`/painel/estudio/:id/perfil` 👑)
- [ ] T076 [P] [US1] Implement `frontend/src/features/studio/StudioArtistsPage.tsx` (`/painel/estudio/:id/tatuadores`: invite by slug, pending invites with countdown + cancel, active artists with 7 weekday checkboxes, end)
- [ ] T077 [P] [US1] Implement `frontend/src/features/studio/SchedulePage.tsx` (`/painel/estudio/:id/horarios`: weekly windows editor, date exceptions, one-off windows, booking window; seats read-only for members, editable for owner; 409 messages)
- [ ] T078 [P] [US1] Implement `frontend/src/features/artist/ArtistStudiosPage.tsx` (`/painel/tatuador/estudios`: invites with countdown, accept/decline, active studios with read-only weekdays)
- [ ] T079 [P] [US1] Implement public studio page `frontend/src/features/discovery/StudioPage.tsx` (`/estudios/:slug`, stacked on mobile, cards on desktop, empty state without artists)
- [ ] T080 [US1] Write e2e `frontend/e2e/us1-studio-setup.spec.ts` covering quickstart V1 in both viewports

**Checkpoint**: US1 funcional e testável sozinha

---

## Phase 4: User Story 2 - Cliente agenda sessão escolhendo estúdio e tatuador (Priority: P1)

**Goal**: cliente (visitante ou conta) envia pedido sem reserva com data, faixa e detalhes;
tatuador aprova definindo início e duração (regras de ocupação sob lock), recusa, ou o
pedido expira; remarcação, cancelamento, falta, lembrete e resumo diário; link do visitante
recuperável e aviso de e-mail não entregue.

**Independent Test**: quickstart.md V2–V6, V12 e V14.

### Tests for User Story 2

- [ ] T081 [P] [US2] Write unit tests in `backend/tests/unit/scheduling/test_booking_rules.py`: `expires_at = min(requested + 48h, start of requested window)` computed from `requested_date` + `requested_window_start` in the studio timezone (same-day request at 10:00 for window 14:00–18:00 → expires 14:00, never born expired), recomputed on reschedule (FR-011b), transitions incl. reschedule (clears interval, `decided_at`, `reminder_sent_at`), no-show mark/unmark window `[starts_at, starts_at + 7d)`, who may approve/decline/cancel/reschedule/mark, cancel of confirmed after start → `booking_started`, request details validation, self-booking detection
- [ ] T082 [P] [US2] Write unit tests in `backend/tests/unit/scheduling/test_approval_rules.py`: duration multiple of 30 ≥ 30, start on requested date and in the future, interval fully inside union of contiguous windows (spanning two windows ok, gap → rejected), per-window capacity peak, studio seats peak with overlapping windows and different starts
- [ ] T083 [P] [US2] Write integration tests in `backend/tests/integration/test_bookings_create.py`: guest multipart with 2 images → 201 pending with `guest_access_url`, `booking_received` (client) + `booking_requested` (artist) + studio notification + outbox rows, **no capacity consumed**; logged-in account uses own data; `self_booking`; inactive affiliation, unpublishable, outside window or weekday, blocked date or window not on date → 409 `slot_unavailable`; third pending by email or by phone in other format → 409 `pending_limit_reached`; invalid image → 400 `invalid_image` and no orphan objects
- [ ] T084 [P] [US2] Write integration tests in `backend/tests/integration/test_bookings_approval.py`: `free-intervals` reflects windows, confirmed load and artist sessions; approve spanning two windows → confirmed; outside windows/past → `slot_unavailable`; seats or window capacity exhausted → `slot_full`; artist overlap in another studio → `artist_unavailable`; affiliation ended after request → `affiliation_inactive`; client and studio notified with start and duration
- [ ] T085 [P] [US2] Write concurrency tests in `backend/tests/integration/test_bookings_concurrency.py`: 20 concurrent approvals (different artists) competing for the last seat with overlapping intervals of different starts → exactly one success; same artist approved in two studios simultaneously → one success; 10 concurrent requests with same email in different studios → at most two 201
- [ ] T086 [P] [US2] Write integration tests in `backend/tests/integration/test_bookings_lifecycle.py`: decline, cancel pending anytime, cancel confirmed before start (frees interval) and after start → 409, studio member cancel audited with `cancelled_by_user_id`, reschedule confirmed → pending with new preference and freed interval, reschedule blocked by pending limit leaves booking unchanged, no-show mark/unmark by artist and by studio member (audited), outside window → `no_show_window_closed`, guest token view/cancel/reschedule, reference images only for parties and guest token, `slot_has_bookings` and `seats_below_bookings` now enforced against confirmed sessions
- [ ] T087 [P] [US2] Write integration tests in `backend/tests/integration/test_booking_jobs.py` (FakeClock): expire pending at 48h and at the start of the requested window (incl. same-day request with min notice 0h), complete confirmed 7 days after start, client reminder once at T−24h (not for sessions confirmed within 24h; reschedule re-arms), artist digest at 18:00 Brasília grouping both studios in their timezones, once per day (`job_run`), skipping artists without sessions
- [ ] T088 [P] [US2] Write integration tests in `backend/tests/integration/test_guest_access.py`: `POST /guest/resend-links` always 204, sends one `guest_links` e-mail with new links for all active bookings of that e-mail and invalidates old tokens, no e-mail when none; bounce scan (fake Gmail inbox) marks the client notification failed and booking returns `client_email_undelivered: true` only to parties; verifying an account's e-mail links guest bookings (FR-002a); `GET /me/bookings` and `GET /studios/{id}/bookings?artist=` (FR-013)

### Implementation for User Story 2

- [ ] T089 [US2] Create `Booking` (all fields in data-model.md: `requested_date`, requested window, nullable `starts_at`/`ends_at`, `no_show_*`, status incl. `no_show`) and `BookingReferenceImage` in `backend/app/modules/scheduling/models.py` and migration `backend/migrations/versions/0006_bookings.py` with `EXCLUDE ... WHERE status = 'confirmed'`, interval-nullity CHECK per data-model.md (confirmed/completed/no_show ⇒ both set; pending/declined/expired ⇒ both null; cancelled ⇒ both null or both set) and partial indexes
- [ ] T090 [P] [US2] Implement booking domain in `backend/app/modules/scheduling/domain/booking.py` (`BookingStatus`, `compute_expires_at`, `transition(booking, command: BookingCommand)`, `RequestDetails` validation) to pass T081
- [ ] T091 [P] [US2] Implement approval rules in `backend/app/modules/scheduling/domain/approval.py` (`validate_interval(proposal: IntervalProposal, context: ApprovalContext)` covering duration, date, windows union, per-window capacity peak and seats peak via sweep line) to pass T082
- [ ] T092 [US2] Implement `BookingRepository` in `backend/app/modules/scheduling/booking_repository.py`: confirmed load for studio/date, artist confirmed intervals, pending count by normalized email/phone, `lock_client(keys: ClientLockKeys)` and `lock_studio(studio_id)` advisory locks (client keys sorted), expire-by-id, guest token lookup/rotation, list queries per side; map exclusion-constraint `IntegrityError` → `artist_unavailable`
- [ ] T093 [US2] Implement reference image storage in `backend/app/modules/scheduling/references.py`: process up to 3 images with `process_image` before the transaction, upload to `booking-refs/{booking_id}/{n}.webp`, cleanup prefix on failure, stream for reading
- [ ] T094 [US2] Implement `BookingService.request(command: RequestBookingCommand)` in `backend/app/modules/scheduling/booking_service.py`: publishability, affiliation, self-booking, date in window + weekday + window exists on date, client locks + pending limit, insert pending with images, guest token, notifications (`booking_received` with link, `booking_requested`, studio)
- [ ] T095 [US2] Implement `BookingService.free_intervals` and `BookingService.approve(command: ApproveCommand)` in `backend/app/modules/scheduling/booking_service.py`: studio lock, reject with `invalid_transition` if `expires_at <= now` even before the periodic task runs, reload affiliation/publishability, `validate_interval`, set interval and `confirmed`, notifications with start and duration
- [ ] T096 [US2] Implement `BookingService.decline|cancel|reschedule|mark_no_show|unmark_no_show|get|list_*` in `backend/app/modules/scheduling/booking_service.py`; `reschedule` reuses the request validation path (client locks, pending limit excluding itself) and clears the interval; studio-side cancel and no-show audited
- [ ] T097 [US2] Wire real `ConfirmedLoadProvider` and `ArtistBusyProvider` from `BookingRepository` into scheduling and studios services in `backend/app/modules/scheduling/dependencies.py`; implement `GuestBookingLinker` used by account verification
- [ ] T098 [US2] Implement `booking_party` dependency (client / artist / studio member sides, optional `side`) in `backend/app/modules/scheduling/dependencies.py` and routes in `backend/app/modules/scheduling/booking_router.py`: `POST /bookings` (multipart, returns `guest_access_url` for guests), `GET /me/bookings`, `GET /studios/{id}/bookings`, `GET /bookings/{id}`, `free-intervals`, `approve`, `decline`, `cancel`, `reschedule`, `no-show` (POST/DELETE), `references/{n}`, all `/guest/bookings/{token}/...` routes and `POST /guest/resend-links`; contact/details/`client_email_undelivered` only for parties
- [ ] T099 [US2] Implement periodic tasks in `backend/app/modules/scheduling/tasks.py`: `expire_pending_bookings` (1 min, `expires_at <= now`), `complete_past_bookings` (hourly, 7 days after start), `send_client_reminders` (5 min), `send_artist_daily_digest` (crontab 18:00 `America/Sao_Paulo`, `job_run` guard); register in `backend/app/celery_app.py`
- [ ] T100 [US2] Implement Gmail bounce scan task `notifications.scan_bounces` (every 10 min, only when `MAIL_BACKEND=gmail`) in `backend/app/modules/notifications/bounces.py` and `tasks.py`: query mailer-daemon DSNs, parse original `Message-ID`, set `email_failed_at` + `delivery_failure_reason`, apply label `tatuagem-processado`; DSN parsing covered by unit tests with fixture messages in `backend/tests/unit/notifications/test_bounces.py`
- [ ] T101 [US2] Implement public artist page base `frontend/src/features/portfolio/ArtistPage.tsx` (`/tatuadores/:slug`: profile, styles, active studios with "Agendar", empty state) and backend `GET /artists/{slug}` in `backend/app/modules/artists/router.py` (publishable only, FR-016)
- [ ] T102 [US2] Implement booking wizard in `frontend/src/features/booking/BookingWizard.tsx` (`/agendar/:studioSlug`): artist/studio choice, calendar with offered dates and window choice (text "o tatuador define o horário exato e a duração ao aprovar"), request details with up to 3 image previews and camera on mobile, client data (guest form or account), confirmation with `CopyLinkField` for guests; 409 handling incl. `pending_limit_reached`
- [ ] T103 [P] [US2] Implement `frontend/src/features/booking/RescheduleDialog.tsx` (reuses wizard date/window step) and `frontend/src/features/booking/GuestBookingPage.tsx` (`/agendamentos/acesso/:token`: status, requested date/window or confirmed start+duration, details, images, reschedule, cancel)
- [ ] T104 [P] [US2] Implement `frontend/src/features/booking/ResendLinkPage.tsx` (`/agendamentos/reenviar-link`, neutral message) and `frontend/src/features/booking/MyBookingsPage.tsx` (`/painel/agendamentos`: client-side bookings, reschedule and cancel per rules)
- [ ] T105 [P] [US2] Implement `frontend/src/features/artist/ArtistRequestsPage.tsx` (`/painel/tatuador/pedidos`: pending with expiry countdown, requested date/window, details, zoomable images, "remarcado" badge, decline) and `frontend/src/features/artist/ApproveDialog.tsx` (free-interval picker on 30-min grid, start + duration, 409 messages)
- [ ] T106 [P] [US2] Implement `frontend/src/features/studio/StudioAgendaPage.tsx` (`/painel/estudio/:id/agenda`: confirmed sessions and pending requests, week view desktop / day view mobile, filter by artist, cancel, "Não compareceu" with undo, "e-mail do cliente não entregue" badge with highlighted phone)
- [ ] T107 [US2] Write e2e `frontend/e2e/us2-booking.spec.ts` covering quickstart V2, V5, V6, V12 and V14 (steps 1–2) in both viewports (reads e-mails from Mailpit)

**Checkpoint**: US1 + US2 entregam o fluxo completo de pedido e aprovação

---

## Phase 5: User Story 7 - Privacidade e exclusão de conta (Priority: P1)

**Goal**: dados de clientes restritos às partes, anonimização após 12 meses, exclusão de
estúdio e de conta.

**Independent Test**: quickstart.md V11.

### Tests for User Story 7

- [ ] T108 [P] [US7] Write integration tests in `backend/tests/integration/test_privacy.py`: every public route (`/studios/{slug}`, `/artists/{slug}`, `/search/*` when present, availability, portfolio when present) never returns client e-mail, phone, description or images; anonymization at 12 months (after `ends_at`, or after the end of `requested_date` when never confirmed: declined, expired, cancelled while pending) clears contact, normalized fields, description, guest token, images and notification e-mails while keeping date/studio/artist/status
- [ ] T109 [P] [US7] Write integration tests in `backend/tests/integration/test_deletions.py`: owner deletes studio (future sessions and pending requests cancelled with notifications, affiliations ended, members removed, owner keeps account and artist profile); account deletion cascades (owned studio deleted, memberships removed, artist profile removed with content cleanup hook, client-side bookings cancelled, sessions deleted, account anonymized, artist `display_name` kept in others' history); wrong password → 400 `invalid_password`; purge of published outbox rows older than 7 days

### Implementation for User Story 7

- [ ] T110 [US7] Implement periodic tasks `scheduling.anonymize_old_bookings` (hourly) in `backend/app/modules/scheduling/tasks.py` and `notifications.purge_published_outbox` (daily, > 7 days) in `backend/app/modules/notifications/tasks.py`; register in `backend/app/celery_app.py`
- [ ] T111 [US7] Implement `StudioService.delete(command: DeleteStudioCommand)` in `backend/app/modules/studios/service.py` and `DELETE /studios/{id}` (owner, password) in `backend/app/modules/studios/router.py`
- [ ] T112 [US7] Implement account deletion in `backend/app/modules/accounts/deletion.py` composing studio deletion, membership removal, artist profile removal (via `ArtistContentCleaner` protocol; no-op until US3 registers portfolio cleanup), client-side cancellations, session/token removal and anonymization; route `DELETE /me` in `backend/app/modules/accounts/router.py`
- [ ] T113 [P] [US7] Add "Excluir estúdio" to `frontend/src/features/studio/StudioProfilePage.tsx` and "Excluir conta" (impact summary from `/auth/me` areas + password) to `frontend/src/features/account/AccountPage.tsx`
- [ ] T114 [US7] Write e2e `frontend/e2e/us7-privacy.spec.ts` covering quickstart V11 steps 2–3 in both viewports

**Checkpoint**: MVP pronto para dados reais (LGPD)

---

## Phase 6: User Story 3 - Tatuador publica portfólio (Priority: P2)

**Goal**: tatuador publica, edita, ordena e remove trabalhos; visitantes veem a grade e o
trabalho ampliado com atalho para agendar.

**Independent Test**: quickstart.md V7.

### Tests for User Story 3

- [ ] T115 [P] [US3] Write integration tests in `backend/tests/integration/test_portfolio.py`: create with ≥ 1 style (0 → 400), invalid image → 400 `invalid_image`, public list paginated by position, edit/reorder/delete only by owner artist (others 403), delete removes storage prefix, account deletion removes all portfolio objects, unpublishable artist → 404 publicly

### Implementation for User Story 3

- [ ] T116 [P] [US3] Create `PortfolioItem` + `PortfolioItemStyle` in `backend/app/modules/artists/portfolio/models.py` and migration `backend/migrations/versions/0007_portfolio.py`
- [ ] T117 [US3] Implement `PortfolioRepository` + `PortfolioService` in `backend/app/modules/artists/portfolio/repository.py` and `service.py` (`process_image` → thumb 600 + full 1600 in `portfolio`, reorder, delete with storage cleanup) and register the `ArtistContentCleaner` implementation used by account deletion
- [ ] T118 [US3] Implement routes in `backend/app/modules/artists/portfolio/router.py`: `GET /artists/{slug}/portfolio`, `POST /me/artist/portfolio`, `PATCH /me/artist/portfolio/{id}`, `DELETE /me/artist/portfolio/{id}`
- [ ] T119 [P] [US3] Implement `frontend/src/features/portfolio/PortfolioGrid.tsx` (CSS `columns` masonry 2/3/4–5 cols, width/height attributes, `loading="lazy"`, `alt` = title, infinite scroll)
- [ ] T120 [US3] Add the grid to `frontend/src/features/portfolio/ArtistPage.tsx` and implement `frontend/src/features/portfolio/WorkDetail.tsx` (`/tatuadores/:slug/trabalhos/:id`: full-screen swipe on mobile, modal on desktop, "Agendar")
- [ ] T121 [P] [US3] Implement `frontend/src/features/portfolio/ManagePortfolioPage.tsx` (`/painel/tatuador/portfolio`: upload with preview, edit, delete, reorder by drag on desktop and buttons on mobile)
- [ ] T122 [US3] Add `--portfolio N` to `backend/app/seed.py` and write e2e `frontend/e2e/us3-portfolio.spec.ts` covering V7 steps 1–2 in both viewports

**Checkpoint**: portfólio público funcional

---

## Phase 7: User Story 4 - Tatuador gerencia vínculos e agenda consolidada (Priority: P2)

**Goal**: tatuador vê num só lugar as sessões e pedidos de todos os estúdios, marca faltas e
encerra vínculos.

**Independent Test**: quickstart.md V8 passo 1 e V4 passo 4.

### Tests for User Story 4

- [ ] T123 [P] [US4] Write integration tests in `backend/tests/integration/test_artist_agenda.py`: `GET /me/artist/bookings` returns sessions and requests from two studios with studio summary, `from`/`to`/`studio` filters, artist ends affiliation (studio notified, sessions kept, pending ones no longer approvable, new request → 409 `affiliation_inactive`)

### Implementation for User Story 4

- [ ] T124 [US4] Implement `GET /me/artist/bookings` with filters and embedded `StudioSummary` in `backend/app/modules/scheduling/booking_router.py` and `booking_service.py` (pass T123)
- [ ] T125 [P] [US4] Implement `frontend/src/features/artist/ArtistAgendaPage.tsx` (`/painel/tatuador/agenda`: grouped by day, studio badge/color, filter by studio, status chips, cancel until start, "Não compareceu" with undo within 7 days)
- [ ] T126 [P] [US4] Add "encerrar vínculo" with confirmation to `frontend/src/features/artist/ArtistStudiosPage.tsx`
- [ ] T127 [US4] Write e2e `frontend/e2e/us4-artist-agenda.spec.ts` covering V8 step 1 and V4 step 4 in both viewports

**Checkpoint**: tatuador atua em vários estúdios com visão unificada

---

## Phase 8: User Story 6 - Equipe do estúdio (Priority: P2)

**Goal**: dono convida e remove membros e consulta o registro de ações (acesso por nível e
auditoria já existem desde a Phase 2).

**Independent Test**: quickstart.md V10.

### Tests for User Story 6

- [ ] T128 [P] [US6] Write integration tests in `backend/tests/integration/test_team.py`: owner invites existing and new e-mail (`member_invited` via outbox), accept requires logged account with same verified e-mail (other e-mail → 409 `invitation_not_valid`), owner cannot be member, invite expires in 7 days, cancel invite, remove member (audited, access lost immediately, account and other studios unaffected), audit log with actor/period filters only for owner, audit purge after 12 months

### Implementation for User Story 6

- [ ] T129 [US6] Implement `TeamService` in `backend/app/modules/studios/team_service.py`: invite, cancel invite, accept by token, list members + pending invites, remove member; all audited
- [ ] T130 [US6] Implement routes in `backend/app/modules/studios/team_router.py`: `/studios/{id}/members` (GET, DELETE `{user_id}`), `/studios/{id}/member-invites` (POST, DELETE `{invite_id}`), `POST /member-invites/{token}/accept`, `GET /studios/{id}/audit-log` (`studio_access(owner)`)
- [ ] T131 [P] [US6] Implement periodic tasks `studios.purge_audit_log` (hourly, > 12 months) and `studios.expire_member_invites` in `backend/app/modules/studios/tasks.py`; register in `backend/app/celery_app.py`
- [ ] T132 [P] [US6] Implement `frontend/src/features/studio/TeamPage.tsx` (`/painel/estudio/:id/equipe` 👑: members, pending invites, invite by e-mail, remove)
- [ ] T133 [P] [US6] Implement `frontend/src/features/studio/AuditLogPage.tsx` (`/painel/estudio/:id/registro` 👑: list with member and period filters, readable change summaries)
- [ ] T134 [P] [US6] Implement `frontend/src/features/auth/TeamInvitePage.tsx` (`/convite-equipe/:token`: logged → accept; not logged → register with prefilled e-mail then accept)
- [ ] T135 [US6] Write e2e `frontend/e2e/us6-team.spec.ts` covering quickstart V10 in both viewports

**Checkpoint**: equipe do estúdio funcional

---

## Phase 9: User Story 5 - Descoberta de estúdios e tatuadores (Priority: P3)

**Goal**: busca por nome, cidade e estilo; estilo do estúdio derivado dos vínculos ativos.

**Independent Test**: quickstart.md V8 passos 2–3.

### Tests for User Story 5

- [ ] T136 [P] [US5] Write integration tests in `backend/tests/integration/test_search.py`: artists by style/city (accent-insensitive)/partial name, studios by name/city/style (derived from active affiliations; disappears when the only artist of the style leaves), unpublishable or deleted never returned, no client data, pagination

### Implementation for User Story 5

- [ ] T137 [US5] Create migration `backend/migrations/versions/0008_search_indexes.py` (GIN trigram on `artist_profile.display_name` and `studio.name`; btree on `city_normalized`)
- [ ] T138 [US5] Implement `SearchRepository` + `SearchService` + routes in `backend/app/modules/discovery/repository.py`, `service.py`, `router.py`: `GET /search/artists` (with `cover_thumb_url`), `GET /search/studios` (`style` via `EXISTS` on active affiliation + artist style)
- [ ] T139 [US5] Implement `frontend/src/features/discovery/SearchPage.tsx` (`/`: tabs Tatuadores/Estúdios, text, city, style chips, bottom sheet on mobile, sidebar on desktop, filters in URL)
- [ ] T140 [US5] Write e2e `frontend/e2e/us5-search.spec.ts` covering V8 steps 2–3 in both viewports

**Checkpoint**: todas as histórias funcionando

---

## Phase 10: Polish & Cross-Cutting Concerns

- [ ] T141 [P] Write performance check `frontend/e2e/perf-portfolio.spec.ts`: with `seed --portfolio 100` and Fast 4G throttling, grid visible in < 2 s (SC-004)
- [ ] T142 [P] Write e2e `frontend/e2e/responsive.spec.ts`: no horizontal scroll, nav switch, usable area switcher and approval interval picker on every route in both viewports (quickstart V15)
- [ ] T143 [P] Write `frontend/e2e/a11y.spec.ts` with `@axe-core/playwright` on main routes and fix contrast, labels and focus states in `frontend/src/components/`
- [ ] T144 Security hardening in `infra/nginx/nginx.conf`: `limit_req` on login, password reset request, verification resend, `POST /api/v1/bookings`, `POST /api/v1/guest/resend-links` and member invites; production config with TLS and `Secure` cookies
- [ ] T145 [P] Add worker/relay observability in `backend/app/celery_app.py` and `backend/app/outbox_relay.py`: structured logs with `task_id`/outbox id, log alert on `email.dlq` arrivals, on `MailAuthError` and when `mail_sent_today` passes 80% of the daily Gmail limit
- [ ] T146 [P] Write project `README.md` (overview, stack, outbox → RabbitMQ → Celery diagram, how to run, Gmail OAuth setup with "In production" warning, links to quickstart and specs)
- [ ] T147 Run full quickstart.md validation (V1–V15, incl. V4b) and fix discrepancies

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)** → **Foundational (Phase 2)**: bloqueia tudo
- **US1 (Phase 3)**: depende da Phase 2
- **US2 (Phase 4)**: depende de US1 (estúdio, vínculos, faixas e datas oferecidas)
- **US7 (Phase 5)**: depende de US2 (agendamentos a anonimizar/cancelar). A limpeza do
  portfólio na exclusão de conta entra quando US3 registra o `ArtistContentCleaner` (T117)
- **US3 (Phase 6)**: depende só da Phase 2; se US2 ainda não criou `ArtistPage.tsx` (T101),
  T120 cria a página
- **US4 (Phase 7)**: depende de US2
- **US6 (Phase 8)**: depende de US1 (estúdio); usa `studio_access` e `AuditTrail` da Phase 2
- **US5 (Phase 9)**: depende da Phase 2; filtro de estúdio por estilo precisa de vínculos
  (US1); `cover_thumb_url` nulo até US3
- **Polish (Phase 10)**: depois das fases desejadas

### Within Each Phase

- Testes primeiro (devem falhar) → models → migração → domínio → repositório/serviço → rotas
  → tasks → frontend → e2e
- `domain/` não importa SQLAlchemy, FastAPI nem Celery (Princípio III)
- Providers entre módulos (`AreasProvider`, `ConfirmedLoadProvider`, `ArtistBusyProvider`,
  `GuestBookingLinker`, `ArtistContentCleaner`) começam como no-op e recebem a implementação
  real na fase que cria os dados (T097, T117)
- A criação de pedido não trava o estúdio; só a aprovação (T095) usa `lock_studio`

### Parallel Opportunities

- Setup: T003–T005, T007, T008 após T001/T002
- Foundational: T011–T017 juntos; T019 com T018; T030/T031 com T029; T042/T043 com T041;
  T050–T055 após T049
- US1: T056–T061 (testes); T064/T065; T067/T068; telas T075–T079
- US2: T081–T088 (testes); T090/T091; telas T103–T106
- Em paralelo ao caminho crítico US1 → US2 → US7 → US4: US3, depois US6 e US5

---

## Parallel Example: User Story 2

```bash
# Testes de US2 juntos:
Task: "Booking rules in backend/tests/unit/scheduling/test_booking_rules.py"
Task: "Approval rules in backend/tests/unit/scheduling/test_approval_rules.py"
Task: "Create tests in backend/tests/integration/test_bookings_create.py"
Task: "Approval tests in backend/tests/integration/test_bookings_approval.py"
Task: "Concurrency tests in backend/tests/integration/test_bookings_concurrency.py"
Task: "Lifecycle tests in backend/tests/integration/test_bookings_lifecycle.py"
Task: "Job tests in backend/tests/integration/test_booking_jobs.py"
Task: "Guest access tests in backend/tests/integration/test_guest_access.py"

# Domínio puro de US2 junto:
Task: "Booking domain in backend/app/modules/scheduling/domain/booking.py"
Task: "Approval rules in backend/app/modules/scheduling/domain/approval.py"

# Telas de US2 juntas (após as rotas):
Task: "RescheduleDialog/GuestBookingPage", "ResendLinkPage/MyBookingsPage", "ArtistRequestsPage/ApproveDialog", "StudioAgendaPage"
```

---

## Implementation Strategy

### MVP First

1. Phase 1 → Phase 2 (núcleo, mensageria/Gmail, contas, acesso, auditoria)
2. Phase 3 (US1) → **validar** V1
3. Phase 4 (US2) → **validar** V2–V6, V12, V14: primeiro pedido aprovado
4. Phase 5 (US7) → **validar** V11 → **MVP pronto para dados reais**

### Incremental Delivery

1. Setup + Foundational → base pronta
2. US1 → demo para estúdios
3. US2 + US7 → **lançamento MVP**
4. US3 (portfólio) → US4 (agenda consolidada) → US6 (equipe) → US5 (busca)
5. Polish

### Parallel Team Strategy

- Dev A: US1 → US2 → US7 → US4
- Dev B: US3 → US6 → US5 → Polish

---

## Notes

- [P] = arquivos diferentes, sem dependência pendente
- Commit após cada tarefa ou grupo lógico com commits Angular (`feat(scheduling): ...`,
  `test(bookings): ...`); refatorações em commits `refactor` separados (constituição VI)
- Pare em cada checkpoint para validar de forma independente
- Backup automatizado está fora da v1 (spec, Assumptions); nenhuma tarefa o implementa
- O envio real pelo Gmail exige o refresh token gerado por T019 e o app OAuth em
  "In production"; testes e CI usam Mailpit e `FakeMailer`
