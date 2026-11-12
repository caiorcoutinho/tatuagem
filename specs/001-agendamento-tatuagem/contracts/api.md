# Contrato da API REST

**Base**: `/api/v1` (via nginx, mesmo domínio do frontend) · **Formato**: JSON (exceto upload
multipart) · **Fonte de verdade em runtime**: OpenAPI gerado pelo FastAPI em
`/api/v1/openapi.json`; os tipos do frontend são gerados a partir dele.

## Convenções

- **Auth**: cookie de sessão `session` (HttpOnly). Coluna "Auth": `—` público,
  `any` qualquer usuário logado, `studio` / `artist` / `client` papel exigido,
  `guest-token` token do link de acesso no path.
- **Erros**: `{"error": {"code": "<snake_case>", "message": "<texto pt-BR>", "fields"?: {...}}}`
  - `400` dados inválidos (`validation_error`) · `401` `unauthenticated` · `403` `forbidden`
  - `404` `not_found` · `409` conflito de regra de negócio (códigos abaixo)
- **Paginação**: `?cursor=<opaco>&limit=<1..50>` → `{"items": [...], "next_cursor": "..."|null}`
- **Datas**: ISO 8601 com offset; datas locais (`date`, `time`) sem fuso, interpretadas no
  fuso do estúdio.

### Códigos de conflito (409)
| Código | Quando |
|---|---|
| `email_taken` | cadastro com e-mail existente |
| `affiliation_exists` | convite para par com vínculo `invited`/`active` |
| `affiliation_inactive` | agendamento com tatuador sem vínculo ativo no estúdio |
| `slot_unavailable` | horário inexistente, passado ou bloqueado |
| `slot_full` | horário sem vagas |
| `artist_unavailable` | tatuador já tem agendamento ativo sobreposto (qualquer estúdio) |
| `slot_has_bookings` | remover avulso/reduzir capacidade abaixo dos agendamentos ativos |
| `invalid_transition` | mudança de status não permitida (ex.: aprovar agendamento expirado) |

---

## Auth e conta

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/auth/register` | — | `{email, password, role, profile: {...}}` → `201 {user}` + cookie. `profile` varia por role (ver perfis). Senha ≥ 8 caracteres |
| POST | `/auth/login` | — | `{email, password}` → `200 {user}` + cookie |
| POST | `/auth/logout` | any | → `204` |
| GET | `/auth/me` | any | → `{user: {id, email, role, email_verified}, profile}` |
| POST | `/auth/verify-email` | — | `{token}` → `204`; para `client`, associa agendamentos de visitante com o mesmo e-mail (FR-002a) |

## Perfis

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/studios/{slug}` | — | → `{studio, artists: [ArtistSummary]}` (só vínculos ativos) |
| PATCH | `/me/studio` | studio | `{name?, description?, address?, city?, phone?, timezone?}` → `{studio}` |
| GET | `/artists/{slug}` | — | → `{artist, styles, studios: [StudioSummary]}` (FR-016) |
| PATCH | `/me/artist` | artist | `{name?, bio?, city?, style_ids?}` → `{artist}` |
| PATCH | `/me/client` | client | `{name?, phone?}` → `{client}` |
| PUT | `/me/photo` | studio, artist | multipart `file` → `{photo_url}` |
| GET | `/styles` | — | → `[{id, slug, name}]` |

## Descoberta (FR-017)

| Método | Rota | Auth | Parâmetros → Resposta |
|---|---|---|---|
| GET | `/search/artists` | — | `q?, city?, style?` + paginação → `[ArtistSummary {slug, name, city, styles, cover_thumb_url}]` |
| GET | `/search/studios` | — | `q?, city?` + paginação → `[StudioSummary {slug, name, city, photo_url}]` |

## Vínculos (FR-004–FR-006)

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/affiliations` | studio | `{artist_slug}` → `201 {affiliation}` (status `invited`) · `409 affiliation_exists` |
| GET | `/me/affiliations` | studio, artist | `status?` → `[Affiliation {id, studio, artist, status, invited_at}]` |
| POST | `/affiliations/{id}/accept` | artist (convidado) | → `{affiliation}` (`active`) |
| POST | `/affiliations/{id}/decline` | artist (convidado) | → `{affiliation}` (`declined`) |
| POST | `/affiliations/{id}/end` | studio ou artist do vínculo | → `{affiliation}` (`ended`) |

## Agenda do estúdio (FR-007, FR-007a, FR-007b)

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/me/schedule/rules` | studio | → `[WeeklyRule {id, weekday, start_time, end_time, capacity}]` |
| PUT | `/me/schedule/rules` | studio | `[WeeklyRule sem id]` → substitui o padrão semanal inteiro · `400` se houver sobreposição |
| PUT | `/me/schedule/overrides/{date}` | studio | `{blocked: true}` ou `{blocked: false, windows: [{start_time, end_time, capacity}]}` → `{override}` · `409 slot_has_bookings` se reduzir capacidade abaixo do ocupado |
| DELETE | `/me/schedule/overrides/{date}` | studio | → `204` (data volta ao padrão) |
| POST | `/me/schedule/one-off-slots` | studio | `{starts_at, ends_at, capacity}` → `201 {slot}` |
| PATCH | `/me/schedule/one-off-slots/{id}` | studio | `{capacity}` → `{slot}` · `409 slot_has_bookings` |
| DELETE | `/me/schedule/one-off-slots/{id}` | studio | → `204` · `409 slot_has_bookings` |

## Disponibilidade (pública)

| Método | Rota | Auth | Parâmetros → Resposta |
|---|---|---|---|
| GET | `/studios/{slug}/availability` | — | `artist` (slug, **obrigatório** — FR-009), `from`, `to` (datas, intervalo ≤ 60 dias) → `[Occurrence {starts_at, ends_at, free}]`, só ocorrências com `free > 0` em que o tatuador está livre · `400` sem `artist` · `409 affiliation_inactive` |

## Agendamentos

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| POST | `/bookings` | — ou client | `{studio_slug, artist_slug, starts_at, ends_at, notes?, client?: {name, email, phone}}` → `201 {booking}` (status `pending`). Visitante envia `client`; cliente logado usa o próprio perfil. Erros: `409 affiliation_inactive`, `slot_unavailable`, `slot_full`, `artist_unavailable` |
| GET | `/me/bookings` | studio, artist, client | `status?, from?, to?, artist?` (filtro do estúdio, FR-013) + paginação → `[Booking]`; tatuador vê todos os estúdios |
| GET | `/bookings/{id}` | parte envolvida | → `{booking}` |
| POST | `/bookings/{id}/approve` | artist do agendamento | → `{booking}` (`confirmed`) · `409 invalid_transition` |
| POST | `/bookings/{id}/decline` | artist do agendamento | → `{booking}` (`declined`) |
| POST | `/bookings/{id}/cancel` | parte envolvida | → `{booking}` (`cancelled`) |
| GET | `/guest/bookings/{token}` | guest-token | → `{booking}` |
| POST | `/guest/bookings/{token}/cancel` | guest-token | → `{booking}` (`cancelled`) |

`Booking = {id, status, starts_at, ends_at, expires_at, studio: StudioSummary, artist: ArtistSummary, client: {name, email, phone}, notes, cancelled_by?}`.
Os dados de contato do cliente só são retornados para o estúdio e o tatuador do agendamento.

## Portfólio (FR-014, FR-015)

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/artists/{slug}/portfolio` | — | paginação → `[PortfolioItem {id, title, description, styles, thumb_url, full_url, width, height}]` |
| POST | `/me/portfolio` | artist | multipart: `file` (JPEG/PNG/WebP ≤ 10 MB), `title`, `description?`, `style_ids[]` (≥ 1) → `201 {item}` · `400 invalid_image` |
| PATCH | `/me/portfolio/{id}` | artist (dono) | `{title?, description?, style_ids?, position?}` → `{item}` |
| DELETE | `/me/portfolio/{id}` | artist (dono) | → `204` |

## Notificações in-app

| Método | Rota | Auth | Corpo → Resposta |
|---|---|---|---|
| GET | `/me/notifications` | any | `unread?` + paginação → `[Notification {id, type, created_at, read_at, booking_id?, affiliation_id?}]` |
| POST | `/me/notifications/read` | any | `{ids: [...]}` ou `{all: true}` → `204` |

## Saúde

| Método | Rota | Auth | Resposta |
|---|---|---|---|
| GET | `/health` | — | `{status: "ok", db: "ok", storage: "ok"}` |
