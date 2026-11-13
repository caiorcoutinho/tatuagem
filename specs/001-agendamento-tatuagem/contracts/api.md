# Contrato da API REST

**Base**: `/api/v1` (via nginx, mesmo domínio do frontend) · **Formato**: JSON (exceto
uploads multipart) · **Fonte de verdade em runtime**: OpenAPI gerado pelo FastAPI em
`/api/v1/openapi.json`; os tipos do frontend são gerados a partir dele.

## Convenções

- **Auth**: cookie de sessão `session` (HttpOnly). A conta não tem papel fixo (research.md
  R19); a coluna "Auth" indica a permissão checada **no recurso**:
  - `—` público · `any` qualquer conta logada · `artist` conta com perfil de tatuador
  - `member` dono **ou** membro do estúdio `{studio_id}` · `owner` dono do estúdio
  - `party` parte do agendamento (cliente, tatuador, ou dono/membro do estúdio)
  - `guest-token` token do link de acesso no path
- **Erros**: `{"error": {"code": "<snake_case>", "message": "<texto pt-BR>", "fields"?: {...}}}`
  - `400` dados inválidos (`validation_error`) · `401` `unauthenticated` · `403` `forbidden`
    (inclui membro removido tentando agir no estúdio) · `404` `not_found` · `409` regra de
    negócio (códigos abaixo)
- **Paginação**: `?cursor=<opaco>&limit=<1..50>` → `{"items": [...], "next_cursor": "..."|null}`
- **Datas**: ISO 8601 com offset; datas locais (`date`, `time`) sem fuso, no fuso do estúdio.
- **Auditoria**: toda rota `member`/`owner` que altera estado grava uma entrada em
  `studio_audit_log` na mesma transação (FR-023).

### Códigos de conflito (409)
| Código | Quando |
|---|---|
| `email_taken` | cadastro com e-mail existente |
| `already_owns_studio` | conta que já é dona de um estúdio tenta criar outro (FR-003) |
| `artist_profile_exists` | conta que já tem perfil de tatuador tenta criar outro |
| `affiliation_exists` | convite para par com vínculo `invited`/`active` |
| `affiliation_inactive` | agendamento com tatuador sem vínculo ativo no estúdio |
| `slot_unavailable` | horário inexistente, passado, bloqueado, fora da janela de antecedência, fora dos dias do tatuador, ou estúdio/tatuador não publicável |
| `slot_full` | horário sem vagas (capacidade ou macas) |
| `artist_unavailable` | tatuador já tem agendamento ativo sobreposto (qualquer estúdio) |
| `self_booking` | tatuador tentando agendar consigo mesmo |
| `slot_has_bookings` | remover avulso/reduzir capacidade abaixo dos agendamentos ativos |
| `seats_below_bookings` | reduzir macas abaixo do pico de agendamentos ativos simultâneos futuros |
| `booking_started` | cancelar ou remarcar agendamento cujo horário já começou |
| `pending_limit_reached` | já existem 2 pedidos pendentes com o mesmo e-mail ou telefone (FR-011c) |
| `invitation_not_valid` | convite (de vínculo ou de membro) expirado, cancelado, já respondido, ou de outro e-mail |
| `invalid_transition` | mudança de status não permitida (ex.: aprovar agendamento expirado) |

---

## Auth e conta

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/auth/register` | — | `{name, email, phone, password}` → `201 {me}` + cookie; envia verificação. Senha ≥ 8 |
| POST | `/auth/login` | — | `{email, password}` → `200 {me}` + cookie |
| POST | `/auth/logout` | any | → `204` |
| GET | `/auth/me` | any | → `Me` (abaixo) |
| PATCH | `/me` | any | `{name?, phone?}` → `Me` |
| POST | `/auth/verify-email/resend` | any | → `204`; reenvia o link (48h) se ainda não verificado |
| POST | `/auth/verify-email` | — | `{token}` → `204`; associa agendamentos de visitante com o mesmo e-mail à conta (FR-002a) |
| POST | `/auth/password-reset/request` | — | `{email}` → sempre `204`; e-mail com link `/redefinir-senha/{token}` (1h) se a conta existir (FR-003a) |
| POST | `/auth/password-reset/confirm` | — | `{token, new_password}` → `204`; encerra todas as sessões · `400 invalid_token` |
| DELETE | `/me` | any | `{password}` → `204`; exclui a conta (FR-020): exclui o estúdio próprio, sai das equipes, exclui o perfil de tatuador, cancela agendamentos futuros com notificação e anonimiza · `400 invalid_password` |

`Me = {id, name, email, phone, email_verified, areas: {artist: ArtistSummary | null, owned_studio: StudioSummary | null, member_of: [StudioSummary]}}`.
O frontend monta o seletor de área a partir de `areas` (FR-003).

## Perfil de tatuador

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/me/artist` | any | `{display_name, bio?, city?, style_ids}` → `201 {artist}` · `409 artist_profile_exists` |
| PATCH | `/me/artist` | artist | `{display_name?, bio?, city?, style_ids?}` → `{artist}` |
| PUT | `/me/artist/photo` | artist | multipart `file` → `{photo_url}` |
| GET | `/artists/{slug}` | — | → `{artist, styles, studios: [StudioSummary]}` (só publicáveis com vínculo ativo, FR-016) |
| GET | `/styles` | — | → `[{id, slug, name}]` |

## Estúdio

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/studios` | any | `{name, address, city, timezone, seats, description?, phone?}` → `201 {studio}` (a conta vira dona) · `409 already_owns_studio` |
| GET | `/studios/{slug}` | — | → `{studio, artists: [ArtistSummary]}` (só publicáveis com vínculo ativo) |
| GET | `/studios/{id}/settings` | member | → `{studio, seats, min_notice_hours, max_advance_days, my_access: "owner"\|"member"}` |
| PATCH | `/studios/{id}` | owner | `{name?, description?, address?, city?, phone?, timezone?, seats?}` → `{studio}` · `409 seats_below_bookings` |
| PATCH | `/studios/{id}/booking-window` | member | `{min_notice_hours?, max_advance_days?}` (0–72, 7–180) → `{studio}` (FR-007e) |
| PUT | `/studios/{id}/photo` | owner | multipart `file` → `{photo_url}` |
| DELETE | `/studios/{id}` | owner | `{password}` → `204` (FR-020a) · `400 invalid_password` |

## Equipe do estúdio (FR-021 a FR-023)

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/studios/{id}/members` | owner | → `[{user_id, name, email, added_at}]` + convites pendentes |
| POST | `/studios/{id}/member-invites` | owner | `{email}` → `201 {invite}`; e-mail com link `/convite-equipe/{token}` (7 dias) |
| DELETE | `/studios/{id}/member-invites/{invite_id}` | owner | → `204` (convite `cancelled`) |
| POST | `/member-invites/{token}/accept` | any | → `{studio}`; exige conta logada com o e-mail convidado e verificado · `409 invitation_not_valid` |
| DELETE | `/studios/{id}/members/{user_id}` | owner | → `204`; acesso removido na hora (FR-021) |
| GET | `/studios/{id}/audit-log` | owner | `actor?, from?, to?` + paginação → `[{actor_name, action, target_type, target_id, changes, created_at}]` |

## Vínculos (FR-004 a FR-007d)

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/studios/{id}/affiliations` | member | `{artist_slug}` → `201 {affiliation}` (`invited`, expira em 24h) · `409 affiliation_exists` |
| GET | `/studios/{id}/affiliations` | member | `status?` → `[Affiliation {id, artist, status, invited_at, expires_at, weekdays}]` |
| POST | `/studios/{id}/affiliations/{aid}/cancel` | member | → `{affiliation}` (`cancelled`) · `409 invitation_not_valid` |
| PUT | `/studios/{id}/affiliations/{aid}/weekdays` | member | `{weekdays: ["mon","wed",...]}` (≥ 1 dia) → `{affiliation}` (FR-007d) |
| POST | `/studios/{id}/affiliations/{aid}/end` | member | → `{affiliation}` (`ended`, `ended_by = studio`) |
| GET | `/me/affiliations` | artist | `status?` → `[Affiliation {id, studio, status, expires_at, weekdays}]` (dias somente leitura) |
| POST | `/me/affiliations/{aid}/accept` | artist (convidado) | → `{affiliation}` (`active`, `weekdays` = todos) · `409 invitation_not_valid` |
| POST | `/me/affiliations/{aid}/decline` | artist (convidado) | → `{affiliation}` (`declined`) · `409 invitation_not_valid` |
| POST | `/me/affiliations/{aid}/end` | artist | → `{affiliation}` (`ended`, `ended_by = artist`) |

## Agenda do estúdio (FR-007 a FR-007c)

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/studios/{id}/schedule/rules` | member | → `[WeeklyRule {id, weekday, start_time, end_time, capacity}]` |
| PUT | `/studios/{id}/schedule/rules` | member | `[WeeklyRule sem id]` → substitui o padrão · `400` se houver sobreposição entre regras ou `capacity > seats` |
| PUT | `/studios/{id}/schedule/overrides/{date}` | member | `{blocked: true}` ou `{blocked: false, windows: [...]}` → `{override}` · `409 slot_has_bookings` |
| DELETE | `/studios/{id}/schedule/overrides/{date}` | member | → `204` |
| POST | `/studios/{id}/schedule/one-off-slots` | member | `{starts_at, ends_at, capacity}` → `201 {slot}` · `400` se `capacity > seats` |
| PATCH | `/studios/{id}/schedule/one-off-slots/{sid}` | member | `{capacity}` → `{slot}` · `409 slot_has_bookings` |
| DELETE | `/studios/{id}/schedule/one-off-slots/{sid}` | member | → `204` · `409 slot_has_bookings` |

## Disponibilidade (pública)

| Método | Rota | Auth | Parâmetros → Resposta |
|---|---|---|---|
| GET | `/studios/{slug}/availability` | — | `artist` (slug, **obrigatório**, FR-009), `from`, `to` → `[Occurrence {starts_at, ends_at, free}]`, recortado pela janela de antecedência, pelos dias do tatuador e pela agenda dele em todos os estúdios · `400` sem `artist` · `409 affiliation_inactive` |

## Agendamentos

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/bookings` | — ou any | **multipart**: `studio_slug`, `artist_slug`, `starts_at`, `ends_at`, `description` (1–1000), `body_location`, `size` (`P`\|`M`\|`G`), `references` (0–3, JPEG/PNG/WebP ≤ 10 MB); visitante envia `client_name`, `client_email`, `client_phone` (conta logada usa os próprios dados) → `201 {booking}` (`pending`) · `400 validation_error`, `400 invalid_image`, `409 affiliation_inactive`, `slot_unavailable`, `slot_full`, `artist_unavailable`, `self_booking`, `pending_limit_reached` |
| GET | `/me/bookings` | any | `status?, from?, to?` + paginação → `[Booking]` em que a conta é cliente (FR-002a) |
| GET | `/me/artist/bookings` | artist | `status?, from?, to?, studio?` + paginação → `[Booking]` de todos os estúdios (FR-013, US4) |
| GET | `/studios/{id}/bookings` | member | `status?, from?, to?, artist?` + paginação → `[Booking]` do estúdio (FR-013) |
| GET | `/bookings/{id}` | party | → `{booking}` |
| POST | `/bookings/{id}/approve` | artist do agendamento | → `{booking}` (`confirmed`) · `409 invalid_transition` |
| POST | `/bookings/{id}/decline` | artist do agendamento | → `{booking}` (`declined`) |
| POST | `/bookings/{id}/cancel` | party | → `{booking}` (`cancelled`, `cancelled_by` conforme o lado de quem cancelou) · `409 booking_started` |
| POST | `/bookings/{id}/reschedule` | cliente do agendamento | `{starts_at, ends_at}` → `{booking}` (`pending`, novo horário) · `409 booking_started`, `slot_unavailable`, `slot_full`, `artist_unavailable`, `pending_limit_reached`; em erro nada muda (FR-011d) |
| GET | `/bookings/{id}/references/{n}` | party | → `image/webp` (streaming, `Cache-Control: private, no-store`) |
| GET | `/guest/bookings/{token}` | guest-token | → `{booking}` |
| POST | `/guest/bookings/{token}/cancel` | guest-token | → `{booking}` · `409 booking_started` |
| POST | `/guest/bookings/{token}/reschedule` | guest-token | igual a `/bookings/{id}/reschedule` |
| GET | `/guest/bookings/{token}/references/{n}` | guest-token | igual a `/bookings/{id}/references/{n}` |

Quando uma conta é parte por mais de um lado (ex.: tatuador que também é membro do
estúdio), o lado usado no cancelamento é o do contexto da rota no frontend (`side` opcional
no corpo: `client`\|`artist`\|`studio`; padrão: o primeiro lado válido nessa ordem).

`Booking = {id, status, starts_at, ends_at, expires_at, studio: StudioSummary, artist: ArtistSummary, client: {name, email, phone}, description, body_location, size, references: [{n, url, width, height}], cancelled_by?, rescheduled_at?}`.
Contato, detalhes e imagens só para as partes (FR-008a, FR-018). Anonimizados (FR-019):
`client = {name: "Cliente anonimizado", email: null, phone: null}`, `description = null`,
`references = []`.

## Portfólio (FR-014, FR-015)

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/artists/{slug}/portfolio` | — | paginação → `[PortfolioItem {id, title, description, styles, thumb_url, full_url, width, height}]` |
| POST | `/me/artist/portfolio` | artist | multipart: `file`, `title`, `description?`, `style_ids[]` (≥ 1) → `201 {item}` · `400 invalid_image` |
| PATCH | `/me/artist/portfolio/{id}` | artist (dono do item) | `{title?, description?, style_ids?, position?}` → `{item}` |
| DELETE | `/me/artist/portfolio/{id}` | artist (dono do item) | → `204` |

## Descoberta (FR-017)

| Método | Rota | Auth | Parâmetros → Resposta |
|---|---|---|---|
| GET | `/search/artists` | — | `q?, city?, style?` + paginação → `[ArtistSummary {slug, display_name, city, styles, cover_thumb_url}]` |
| GET | `/search/studios` | — | `q?, city?, style?` + paginação → `[StudioSummary {slug, name, city, photo_url}]`; `style` casa estúdios com ao menos um tatuador de vínculo ativo naquele estilo |

Só perfis publicáveis aparecem; os demais retornam `404` nas páginas públicas.

## Notificações in-app

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/me/notifications` | any | `unread?` + paginação → notificações pessoais (cliente, tatuador) |
| GET | `/studios/{id}/notifications` | member | `unread?` + paginação → notificações do estúdio (FR-024) |
| POST | `/me/notifications/read` | any | `{ids: [...]}` ou `{all: true}` → `204` |
| POST | `/studios/{id}/notifications/read` | member | `{ids: [...]}` ou `{all: true}` → `204` (leitura compartilhada pela equipe) |

## Saúde

| Método | Rota | Auth | Resposta |
|---|---|---|---|
| GET | `/health` | — | `{status: "ok", db: "ok", storage: "ok", broker: "ok", outbox_backlog: <int>}` (`outbox_backlog` = mensagens não publicadas há mais de 1 min) |
