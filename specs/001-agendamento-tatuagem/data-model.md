# Data Model: Agendamento para Tatuadores e Estúdios

**Feature**: `001-agendamento-tatuagem` | **Date**: 2026-11-12 | **Plan**: [plan.md](./plan.md)

Banco: PostgreSQL 16, extensões `btree_gist` e `pg_trgm`. Todos os ids são UUID; datas/horas
absolutas são `timestamptz` em UTC; horas de regra (padrão semanal) são `time` no fuso do
estúdio. Todas as tabelas têm `created_at` e `updated_at` (omitidos abaixo).

## Diagrama de relacionamentos

```text
user 1─1 studio_profile | artist_profile | client_profile   (exatamente um, por role)
user 1─N session
studio_profile N─N artist_profile        via affiliation (status)
studio_profile 1─N weekly_rule | date_override | one_off_slot
studio_profile 1─N booking N─1 artist_profile
booking N─0..1 user (cliente com conta)
artist_profile 1─N portfolio_item N─N style
artist_profile N─N style                 via artist_style
notification N─0..1 user, N─0..1 booking
```

---

## Contas

### user
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| email | citext | único, obrigatório |
| password_hash | text | Argon2 |
| role | enum `studio` \| `artist` \| `client` | imutável (FR-003: um perfil por conta) |
| email_verified_at | timestamptz? | necessário para associar agendamentos de visitante (FR-002a) |

### session
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK user |
| token_hash | bytea | único; SHA-256 do token do cookie |
| expires_at | timestamptz | 30 dias, renovável |

### studio_profile (Estúdio)
| Campo | Tipo | Regras |
|---|---|---|
| user_id | uuid | PK, FK user (role = studio) |
| slug | text | único, usado na URL pública |
| name | text | obrigatório, 2–80 caracteres |
| description | text? | até 2000 caracteres |
| photo_key | text? | chave no MinIO |
| address | text | obrigatório |
| city | text | obrigatório; `city_normalized` (minúsculas, sem acento) indexado |
| phone | text? | contato |
| timezone | text | IANA (ex.: `America/Sao_Paulo`), obrigatório |

### artist_profile (Tatuador)
| Campo | Tipo | Regras |
|---|---|---|
| user_id | uuid | PK, FK user (role = artist) |
| slug | text | único |
| name | text | obrigatório, 2–80 caracteres; índice trigram |
| bio | text? | até 2000 caracteres |
| photo_key | text? | chave no MinIO |
| city | text? | cidade base; `city_normalized` indexado |

### client_profile (Cliente com conta)
| Campo | Tipo | Regras |
|---|---|---|
| user_id | uuid | PK, FK user (role = client) |
| name | text | obrigatório |
| phone | text | obrigatório |

### style (Estilo de tatuagem)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| slug | text | único (ex.: `fineline`, `old-school`, `realismo`) |
| name | text | nome exibido |

Lista fixa (seed). Relacionada ao tatuador por `artist_style(artist_id, style_id)` e aos
trabalhos por `portfolio_item_style(item_id, style_id)`.

---

## Vínculos

### affiliation (Vínculo)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio_profile |
| artist_id | uuid | FK artist_profile |
| status | enum `invited` \| `active` \| `declined` \| `ended` | |
| invited_at / responded_at / ended_at | timestamptz? | |
| ended_by | enum `studio` \| `artist`? | |

**Regras**
- Índice único parcial: no máximo um vínculo `invited` ou `active` por par
  `(studio_id, artist_id)`; histórico de `declined`/`ended` é mantido.
- Só tatuadores cadastrados podem ser convidados (FR-004).

**Transições**
```text
invited ──aceite do tatuador──▶ active ──estúdio ou tatuador encerra──▶ ended
invited ──recusa do tatuador──▶ declined
```
Encerrar não cancela agendamentos futuros; apenas bloqueia novos (edge case da spec).

---

## Agenda

### weekly_rule (Padrão Semanal)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio_profile |
| weekday | smallint | 0 (segunda) – 6 (domingo) |
| start_time / end_time | time | hora local do estúdio; `start_time < end_time` |
| capacity | smallint | ≥ 1 (FR-007) |

Regras do mesmo estúdio no mesmo dia da semana não podem se sobrepor.

### date_override (exceção por data)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK |
| date | date | data local do estúdio; único por `(studio_id, date)` |
| blocked | boolean | `true` = nenhum horário novo na data |
| windows | jsonb? | quando não bloqueada: lista `[{start_time, end_time, capacity}]` que substitui as regras semanais da data |

### one_off_slot (horário avulso)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK |
| starts_at / ends_at | timestamptz | `starts_at < ends_at`, no futuro |
| capacity | smallint | ≥ 1 |

### Ocorrência (Horário Disponível) — calculada, não persistida
`{studio_id, starts_at, ends_at, capacity, booked, free = capacity − booked}`

Cálculo para um intervalo (máx. 60 dias, só datas futuras):
1. Para cada data, se houver `date_override`: bloqueada → nenhuma ocorrência; senão usa
   `windows`. Sem override → aplica as `weekly_rule` do dia da semana.
2. Acrescenta `one_off_slot` do intervalo.
3. `booked` = agendamentos ativos do estúdio com o mesmo `starts_at`/`ends_at`.
4. Ocorrências com `free = 0` não aparecem para o cliente.

**Regras de edição (edge cases)**
- Excluir `one_off_slot` ou reduzir capacidade (avulso ou override) abaixo de `booked` →
  rejeitado (`409`).
- Editar `weekly_rule` ou bloquear uma data → permitido; agendamentos existentes são mantidos
  (guardam o próprio intervalo), e as ocorrências afetadas deixam de aceitar novos
  agendamentos.

### booking (Agendamento)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio_profile |
| artist_id | uuid | FK artist_profile |
| starts_at / ends_at | timestamptz | snapshot da ocorrência escolhida |
| status | enum `pending` \| `confirmed` \| `declined` \| `expired` \| `cancelled` \| `completed` | |
| requested_at | timestamptz | |
| expires_at | timestamptz | `min(requested_at + 48h, starts_at)` (FR-011b) |
| decided_at | timestamptz? | aprovação/recusa |
| cancelled_at / cancelled_by | timestamptz? / enum `client` \| `artist` \| `studio`? | |
| client_user_id | uuid? | FK user (role = client) quando houver conta |
| client_name / client_email / client_phone | text | sempre preenchidos (snapshot) |
| guest_token_hash | bytea? | SHA-256 do token do link de acesso (visitante) |
| notes | text? | descrição da ideia, até 1000 caracteres |

**Validações na criação** (FR-008, FR-009, FR-007a, FR-010)
- estúdio e tatuador obrigatórios; vínculo `active` entre eles;
- `(starts_at, ends_at)` corresponde a uma ocorrência válida do estúdio com `free > 0`;
- `starts_at` no futuro;
- não existe outro agendamento ativo do mesmo tatuador no mesmo horário (garantido pela
  constraint de exclusão);
- no mesmo horário do estúdio, cada tatuador aparece uma vez (FR-007a; a constraint de
  exclusão já cobre esse caso).

**Constraints e índices**
- `EXCLUDE USING gist (artist_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
  WHERE status IN ('pending','confirmed')`
- índices: `(studio_id, starts_at)`, `(artist_id, starts_at)`, `(status, expires_at)`,
  `(client_email)`.

**Transições** (ativos = `pending` + `confirmed`)
```text
            ┌─ tatuador aprova ──▶ confirmed ─┬─ cliente/tatuador/estúdio cancela ─▶ cancelled
pending ────┼─ tatuador recusa ──▶ declined   └─ horário termina (worker) ─────────▶ completed
            ├─ 48h ou início do horário ─▶ expired
            └─ cliente/tatuador/estúdio cancela ─▶ cancelled
```
`declined`, `expired`, `cancelled` e `completed` são finais e liberam a vaga.
`completed` é aplicado automaticamente pelo worker após `ends_at` (não há ação manual na v1).

---

## Portfólio

### portfolio_item (Trabalho de Portfólio)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| artist_id | uuid | FK artist_profile |
| title | text | obrigatório, até 120 caracteres |
| description | text? | até 2000 caracteres |
| image_key | text | prefixo no MinIO; variantes `{key}/thumb.webp` e `{key}/full.webp` |
| width / height | integer | da variante `full`, para reservar espaço na grade |
| position | integer | ordem na grade (mais recente primeiro por padrão) |

Ao menos um estilo por trabalho (FR-014). Remover o item apaga os objetos no MinIO.

---

## Notificações

### notification
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| recipient_user_id | uuid? | destino in-app (null para visitante) |
| recipient_email | citext | destino do e-mail |
| type | enum `booking_requested` \| `booking_confirmed` \| `booking_declined` \| `booking_expired` \| `booking_cancelled` \| `affiliation_invited` \| `affiliation_ended` | |
| booking_id / affiliation_id | uuid? | referência |
| read_at | timestamptz? | leitura in-app |
| email_sent_at | timestamptz? | outbox: worker envia quando null |
| email_attempts | smallint | máx. 5 |

Criada **na mesma transação** do evento que a gera (FR-012).

| Evento | Destinatários |
|---|---|
| agendamento criado | tatuador (ação), estúdio |
| aprovado | cliente, estúdio |
| recusado / expirado | cliente, estúdio |
| cancelado | as duas outras partes |
| convite de vínculo | tatuador |
| vínculo encerrado | a outra parte |
