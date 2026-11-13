# Data Model: Agendamento para Tatuadores e Estúdios

**Feature**: `001-agendamento-tatuagem` | **Date**: 2026-11-13 (revisão: conta = pessoa) |
**Plan**: [plan.md](./plan.md)

Banco: PostgreSQL 16, extensões `citext`, `btree_gist` e `pg_trgm`. Todos os ids são UUID;
datas/horas absolutas são `timestamptz` em UTC; horas de regra (padrão semanal) são `time`
no fuso do estúdio. Todas as tabelas têm `created_at` e `updated_at` (omitidos abaixo).

## Diagrama de relacionamentos

```text
user 1─0..1 artist_profile                     (perfil de tatuador, opcional)
user 1─0..1 studio (como dono)                 (studio.owner_user_id único)
user N─N studio                                via studio_member (equipe)
user 1─N session | password_reset_token | email_verification_token
studio 1─N studio_member_invite | studio_audit_log
studio N─N artist_profile                      via affiliation (status, weekdays)
studio 1─N weekly_rule | date_override | one_off_slot
studio 1─N booking N─1 artist_profile
booking N─0..1 user (cliente com conta)
booking 1─N booking_reference_image (0 a 3)
artist_profile 1─N portfolio_item N─N style
artist_profile N─N style                       via artist_style
notification N─0..1 user | studio | booking | affiliation
```

**Regra de publicação** (FR-001a, research.md R17), usada em todas as consultas públicas:
- tatuador publicável: `artist_profile.deleted_at IS NULL` e o `user` dono do perfil tem
  `email_verified_at IS NOT NULL AND deleted_at IS NULL`;
- estúdio publicável: `studio.deleted_at IS NULL` e o **dono** é publicável no mesmo sentido.

---

## Contas

### user (Conta: a pessoa)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| name | text | obrigatório, 2–80 caracteres |
| email | citext | único, obrigatório |
| phone | text | obrigatório; `phone_norm` (só dígitos com DDI 55) |
| password_hash | text? | Argon2; nulo após exclusão |
| email_verified_at | timestamptz? | FR-001a; também habilita a associação de agendamentos de visitante (FR-002a) |
| deleted_at | timestamptz? | soft delete (FR-020) |

Sem campo de papel: capacidades vêm das relações (perfil de tatuador, estúdio próprio,
equipes). Conta excluída: `email` → `deleted+{id}@invalid`, `name` → "Conta excluída",
`phone` e `password_hash` nulos; login impossível.

### session
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK user |
| token_hash | bytea | único; SHA-256 do token do cookie |
| expires_at | timestamptz | 30 dias, renovável |

Todas as sessões do usuário são apagadas ao redefinir a senha e ao excluir a conta. Remover
alguém de uma equipe **não** mexe nas sessões (a permissão é checada a cada requisição).

### password_reset_token (FR-003a) e email_verification_token (FR-001a)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK user |
| token_hash | bytea | único; SHA-256 do token enviado por e-mail |
| expires_at | timestamptz | reset: criação + 1h; verificação: criação + 48h |
| used_at | timestamptz? | uso único; ao usar, os demais tokens do mesmo tipo do usuário são invalidados |

### artist_profile (Tatuador)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK user, **único** (no máximo um perfil por conta) |
| slug | text | único, URL pública |
| display_name | text | obrigatório, 2–80 caracteres; índice trigram; mantido no histórico após exclusão |
| bio | text? | até 2000 caracteres |
| photo_key | text? | chave no bucket público |
| city | text? | cidade base; `city_normalized` (minúsculas, sem acento) indexado |
| deleted_at | timestamptz? | exclusão do perfil (junto com a conta) |

### style (Estilo de tatuagem)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| slug | text | único (ex.: `fineline`, `old-school`, `realismo`) |
| name | text | nome exibido |

Lista fixa (seed). Relacionada ao tatuador por `artist_style(artist_id, style_id)` e aos
trabalhos por `portfolio_item_style(item_id, style_id)`.

---

## Estúdio e equipe

### studio (Estúdio)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| owner_user_id | uuid | FK user; índice único parcial `WHERE deleted_at IS NULL` (máx. 1 estúdio por dono, FR-003) |
| slug | text | único, URL pública |
| name | text | obrigatório, 2–80 caracteres; índice trigram; mantido no histórico após exclusão |
| description | text? | até 2000 caracteres |
| photo_key | text? | chave no bucket público |
| address | text | obrigatório |
| city | text | obrigatório; `city_normalized` indexado |
| phone | text? | contato público |
| timezone | text | IANA (ex.: `America/Sao_Paulo`), obrigatório |
| seats | smallint | número de macas, ≥ 1 (FR-007c); só o dono altera |
| min_notice_hours | smallint | 0–72, padrão 24 (FR-007e) |
| max_advance_days | smallint | 7–180, padrão 60 (FR-007e) |
| deleted_at | timestamptz? | exclusão do estúdio (FR-020a) |

Reduzir `seats` abaixo do pico de agendamentos ativos simultâneos futuros → `409
seats_below_bookings`. A conta **não** precisa ter perfil de tatuador para ser dona.

### studio_member (Membro do Estúdio)
| Campo | Tipo | Regras |
|---|---|---|
| studio_id | uuid | FK studio; PK composta |
| user_id | uuid | FK user; PK composta; ≠ `studio.owner_user_id` (checado no serviço) |
| added_at | timestamptz | |

Remover = apagar a linha (FR-021). Uma conta pode ser membro de vários estúdios.

### studio_member_invite
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio |
| email | citext | e-mail convidado; único parcial por estúdio `WHERE status = 'pending'` |
| token_hash | bytea | único |
| status | enum `pending` \| `accepted` \| `cancelled` \| `expired` | |
| expires_at | timestamptz | criação + 7 dias |
| invited_by_user_id | uuid | FK user (dono) |

Aceitar exige conta logada **com o mesmo e-mail** e verificada; recusado se a conta for o
dono ou já for membro (`409 invitation_not_valid`).

### Níveis de acesso ao estúdio
| Ação | Dono | Membro |
|---|---|---|
| Agenda (regras, exceções, avulsos), janela de antecedência | ✅ | ✅ |
| Vínculos (convidar, cancelar convite, encerrar) e dias de atendimento | ✅ | ✅ |
| Agendamentos do estúdio (ver detalhes e contato, cancelar) | ✅ | ✅ |
| Perfil público, número de macas | ✅ | ❌ |
| Membros (convidar, remover), registro de ações, excluir estúdio | ✅ | ❌ |

### studio_audit_log (Registro de Ação, FR-023)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio |
| actor_user_id | uuid | FK user |
| actor_name | text | snapshot do nome na hora da ação |
| action | enum | `schedule_rules_replaced`, `override_set`, `override_removed`, `one_off_created`, `one_off_updated`, `one_off_deleted`, `seats_changed`, `booking_window_changed`, `affiliation_invited`, `affiliation_invite_cancelled`, `affiliation_ended`, `artist_weekdays_changed`, `member_invited`, `member_invite_cancelled`, `member_removed`, `booking_cancelled`, `profile_updated` |
| target_type / target_id | text / uuid? | entidade afetada |
| changes | jsonb | resumo antes/depois (ex.: `{"seats": [3, 2]}`) |
| created_at | timestamptz | índice `(studio_id, created_at)` |

Append-only; gravado na mesma transação da ação (research.md R20). Entradas com mais de 12
meses são apagadas pela task periódica `purge_audit_log` (Celery Beat).

---

## Vínculos

### affiliation (Vínculo)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio |
| artist_id | uuid | FK artist_profile |
| status | enum `invited` \| `active` \| `declined` \| `expired` \| `cancelled` \| `ended` | |
| invited_at / responded_at / ended_at | timestamptz? | |
| expires_at | timestamptz | `invited_at + 24h` (FR-004a) |
| ended_by | enum `studio` \| `artist`? | |
| weekdays | smallint | bitmask seg=1 … dom=64; padrão 127 (todos) ao aceitar (FR-007d); só a gestão do estúdio altera |

**Regras**
- Índice único parcial: no máximo um vínculo `invited` ou `active` por par
  `(studio_id, artist_id)`; histórico dos demais status é mantido, e depois deles o estúdio
  pode convidar de novo.
- Só tatuadores com perfil podem ser convidados (FR-004).
- Alterar `weekdays` não afeta agendamentos existentes; só bloqueia novos nos dias removidos.

**Transições**
```text
invited ──aceite do tatuador──▶ active ──estúdio ou tatuador encerra──▶ ended
invited ──recusa do tatuador──▶ declined
invited ──24h sem resposta (Beat)──▶ expired        (notifica o estúdio)
invited ──estúdio cancela──▶ cancelled
```
Aceitar ou recusar exige `status = invited` **e** `now < expires_at`; senão
`409 invitation_not_valid`. Encerrar não cancela agendamentos futuros; apenas bloqueia novos.

---

## Agenda

### weekly_rule (Padrão Semanal)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio |
| weekday | smallint | 0 (segunda) – 6 (domingo) |
| start_time / end_time | time | hora local do estúdio; `start_time < end_time` |
| capacity | smallint | 1 ≤ capacity ≤ `seats` (FR-007, FR-007c) |

Regras do mesmo estúdio no mesmo dia da semana não podem se sobrepor entre si. Horários de
origens diferentes (padrão, exceção, avulso) **podem** se sobrepor; o limite físico é
garantido por `seats`.

### date_override (exceção por data)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK |
| date | date | data local do estúdio; único por `(studio_id, date)` |
| blocked | boolean | `true` = nenhum horário novo na data |
| windows | jsonb? | quando não bloqueada: `[{start_time, end_time, capacity}]` que substitui as regras semanais da data; sem sobreposição entre si; `capacity ≤ seats` |

### one_off_slot (horário avulso)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK |
| starts_at / ends_at | timestamptz | `starts_at < ends_at`, no futuro |
| capacity | smallint | 1 ≤ capacity ≤ `seats` |

### Ocorrência (Horário Disponível): calculada, não persistida
`{studio_id, starts_at, ends_at, capacity, booked, free}`

Cálculo para um intervalo pedido, recortado pela janela do estúdio
`[now + min_notice_hours, now + max_advance_days]` (FR-007e):
1. Para cada data, se houver `date_override`: bloqueada → nenhuma ocorrência; senão usa
   `windows`. Sem override → aplica as `weekly_rule` do dia da semana.
2. Acrescenta `one_off_slot` do intervalo.
3. Quando a consulta é para um tatuador: descarta ocorrências cujo dia da semana local não
   está em `affiliation.weekdays` (FR-007d) e as que se sobrepõem a agendamentos ativos dele
   em qualquer estúdio (FR-010).
4. `booked` = agendamentos ativos do estúdio com o mesmo `starts_at`/`ends_at`.
5. `peak` = maior número de agendamentos ativos do estúdio simultâneos em algum instante do
   intervalo da ocorrência (qualquer origem).
6. `free = min(capacity − booked, seats − peak)`, mínimo 0 (FR-007c); `free = 0` não aparece.

**Regras de edição (edge cases)**
- Excluir `one_off_slot` ou reduzir capacidade (avulso ou override) abaixo de `booked` →
  `409 slot_has_bookings`.
- Editar `weekly_rule` ou bloquear uma data → permitido; agendamentos existentes são mantidos
  (guardam o próprio intervalo) e as ocorrências afetadas deixam de aceitar novos.

### booking (Agendamento)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| studio_id | uuid | FK studio |
| artist_id | uuid | FK artist_profile |
| starts_at / ends_at | timestamptz | snapshot da ocorrência escolhida |
| status | enum `pending` \| `confirmed` \| `declined` \| `expired` \| `cancelled` \| `completed` | |
| requested_at | timestamptz | |
| expires_at | timestamptz | `min(requested_at + 48h, starts_at)` (FR-011b) |
| decided_at | timestamptz? | aprovação/recusa |
| cancelled_at / cancelled_by | timestamptz? / enum `client` \| `artist` \| `studio`? | |
| cancelled_by_user_id | uuid? | quem da equipe/tatuador/cliente cancelou (auditoria) |
| rescheduled_at | timestamptz? | última remarcação (FR-011d) |
| reminder_sent_at | timestamptz? | lembrete de 24h enviado (FR-012a); zerado ao remarcar |
| client_user_id | uuid? | FK user quando houver conta |
| client_name / client_email / client_phone | text? | snapshot; obrigatórios enquanto `anonymized_at` for nulo |
| client_email_norm / client_phone_norm | text? | normalizados para o limite de pendentes (FR-011c) |
| guest_token_hash | bytea? | SHA-256 do token do link de acesso (visitante) |
| description | text? | 1–1000 caracteres, obrigatória até a anonimização (FR-008a) |
| body_location | enum | `braco`, `antebraco`, `mao`, `ombro`, `costas`, `peito`, `costela`, `abdomen`, `quadril`, `coxa`, `perna`, `panturrilha`, `pe`, `pescoco`, `rosto`, `outro` |
| size | enum `P` \| `M` \| `G` | P até 5 cm, M 5–15 cm, G acima de 15 cm |
| anonymized_at | timestamptz? | FR-019 |

**Validações na criação** (FR-002a, FR-007a, FR-007c, FR-007d, FR-007e, FR-008, FR-008a,
FR-009, FR-010, FR-011c)
- estúdio e tatuador obrigatórios, ambos **publicáveis**; vínculo `active` entre eles;
- o cliente logado não é o próprio tatuador (`artist_profile.user_id ≠ client_user_id`);
- o início está dentro da janela de antecedência e num dia de `affiliation.weekdays`;
- `(starts_at, ends_at)` é uma ocorrência válida com `free > 0` (capacidade e macas, sob lock
  do estúdio — research.md R4);
- descrição, local do corpo e tamanho preenchidos; 0 a 3 imagens válidas;
- menos de 2 `pending` com o mesmo `client_email_norm` **ou** `client_phone_norm`, sob locks
  do cliente;
- sem agendamento ativo sobreposto do mesmo tatuador (constraint de exclusão).

**Constraints e índices**
- `EXCLUDE USING gist (artist_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
  WHERE status IN ('pending','confirmed')`
- índices: `(studio_id, starts_at)`, `(artist_id, starts_at)`, `(status, expires_at)`,
  `(client_user_id)`, `(client_email_norm) WHERE status = 'pending'`,
  `(client_phone_norm) WHERE status = 'pending'`, `(status, starts_at) WHERE reminder_sent_at IS NULL`.

**Transições** (ativos = `pending` + `confirmed`)
```text
            ┌─ tatuador aprova ──▶ confirmed ─┬─ cliente/tatuador/estúdio cancela ─▶ cancelled
pending ────┼─ tatuador recusa ──▶ declined   └─ horário termina (Beat) ───────────▶ completed
            ├─ 48h ou início do horário ─▶ expired
            └─ cliente/tatuador/estúdio cancela ─▶ cancelled

pending | confirmed ──cliente remarca (antes do início)──▶ pending (novo horário)
```
- `declined`, `expired`, `cancelled` e `completed` são finais e liberam a vaga.
- **Cancelar** só com `now < starts_at`, senão `409 booking_started` (FR-011). Pelo lado do
  estúdio, qualquer dono ou membro cancela, e a ação vai para o registro de ações.
- **Remarcar (FR-011d)**: só o cliente (conta ou link), com `now < starts_at` atual, para
  outra ocorrência do mesmo estúdio e tatuador. Atualiza `starts_at`, `ends_at`,
  `requested_at = now`, `expires_at`, `decided_at = null`, `reminder_sent_at = null`,
  `rescheduled_at = now`, `status = pending`. Mesmas validações da criação, excluindo o
  próprio agendamento das contagens; falha → nada muda.
- `completed` é aplicado pela task periódica `complete_past_bookings` após `ends_at`.

**Anonimização (FR-019)**: 12 meses após `ends_at`, `client_name` → "Cliente anonimizado";
`client_email`, `client_phone`, os normalizados, `description` e `guest_token_hash` → nulos;
imagens de referência apagadas; `anonymized_at` preenchido. Data, estúdio, tatuador e status
são mantidos.

### booking_reference_image (FR-008a)
| Campo | Tipo | Regras |
|---|---|---|
| booking_id | uuid | FK booking; PK composta com `position` |
| position | smallint | 1–3 |
| image_key | text | `booking-refs/{booking_id}/{position}.webp` no bucket **privado** |
| width / height | integer | da imagem processada (largura máx. 1600 px) |

Acesso só pelas rotas das partes ou pelo link do visitante (research.md R16).

---

## Portfólio

### portfolio_item (Trabalho de Portfólio)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| artist_id | uuid | FK artist_profile |
| title | text | obrigatório, até 120 caracteres |
| description | text? | até 2000 caracteres |
| image_key | text | prefixo no bucket público; variantes `{key}/thumb.webp` e `{key}/full.webp` |
| width / height | integer | da variante `full`, para reservar espaço na grade |
| position | integer | ordem na grade (mais recente primeiro por padrão) |

Ao menos um estilo por trabalho (FR-014). Remover o item apaga os objetos no MinIO.

---

## Notificações e jobs

### notification
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK |
| recipient_user_id | uuid? | destino in-app pessoal (tatuador, cliente com conta) |
| studio_id | uuid? | destino in-app do estúdio: visível ao dono e aos membros (FR-024) |
| recipient_email | citext | destino do e-mail (para o estúdio: e-mail do dono) |
| type | enum | `booking_requested`, `booking_confirmed`, `booking_declined`, `booking_expired`, `booking_cancelled`, `booking_rescheduled`, `booking_reminder`, `artist_daily_digest`, `affiliation_invited`, `affiliation_expired`, `affiliation_ended`, `member_invited`, `email_verification`, `password_reset` |
| booking_id / affiliation_id | uuid? | referência |
| payload | jsonb? | dados do e-mail (ex.: lista do digest) |
| read_at | timestamptz? | leitura in-app; para notificações de estúdio, leitura compartilhada pela equipe |
| email_sent_at | timestamptz? | preenchido pela task `notifications.send_email`; não nulo = já enviado (idempotência) |
| email_attempts | smallint | incrementado a cada tentativa da task |
| email_failed_at | timestamptz? | tentativas esgotadas; mensagem foi para a dead-letter `email.dlq` |

`booking_reminder`, `artist_daily_digest`, `member_invited`, `email_verification` e
`password_reset` são só e-mail. A notificação e a respectiva `outbox_message`
(`notifications.send_email`, `payload = {notification_id}`) são criadas **na mesma
transação** do evento de negócio (FR-012, research.md R6).

| Evento | Destinatários |
|---|---|
| agendamento criado / remarcado | tatuador (ação), estúdio |
| aprovado | cliente, estúdio |
| recusado / expirado | cliente, estúdio |
| cancelado | as duas outras partes |
| lembrete 24h | cliente (FR-012a) |
| resumo diário | tatuador (FR-012a) |
| convite de vínculo | tatuador |
| convite de vínculo expirado | estúdio |
| vínculo encerrado | a outra parte |
| convite de membro | e-mail convidado |
| verificação / recuperação de senha | a própria conta |

Notificações de agendamentos anonimizados têm `recipient_email` anonimizado também.

### outbox_message (outbox transacional, research.md R6)
| Campo | Tipo | Regras |
|---|---|---|
| id | uuid | PK; também usado como `task_id` do Celery (rastreio e deduplicação) |
| task | text | nome da task Celery (ex.: `notifications.send_email`) |
| queue | text | `email` ou `default` |
| payload | jsonb | argumentos da task; só ids, nunca dados pessoais |
| created_at | timestamptz | gravado na mesma transação da mudança de negócio |
| published_at | timestamptz? | preenchido pelo relay após confirmação do RabbitMQ |
| publish_attempts | smallint | falhas de publicação (broker fora do ar) |

Índice parcial `(created_at) WHERE published_at IS NULL`. O relay lê com
`FOR UPDATE SKIP LOCKED`. Mensagens publicadas há mais de 7 dias são apagadas por uma task
periódica.

### job_run
| Campo | Tipo | Regras |
|---|---|---|
| job | text | PK composta |
| run_date | date | PK composta; garante execução única por dia (resumo diário do tatuador) |
