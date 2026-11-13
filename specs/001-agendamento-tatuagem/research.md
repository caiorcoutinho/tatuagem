# Research: Agendamento para Tatuadores e Estúdios

**Feature**: `001-agendamento-tatuagem` | **Date**: 2026-11-12 | **Plan**: [plan.md](./plan.md)

Stack definida pelo usuário: **React + FastAPI, PostgreSQL, MinIO, nginx na frente**, com UI
responsiva para mobile e desktop. Este documento registra as decisões que completam essa
stack e resolvem os pontos em aberto do Technical Context.

---

## R1. Backend: FastAPI + camada de dados

- **Decision**: Python 3.12, FastAPI, Pydantic v2 (+ pydantic-settings), SQLAlchemy 2.0
  assíncrono com asyncpg, Alembic para migrações. Servido por Uvicorn. `python-multipart`
  é exigido pelo próprio FastAPI para receber `multipart/form-data` (upload de imagens).
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
- **`@hookform/resolvers`**: adaptador oficial que liga os schemas Zod ao React Hook Form;
  sem ele seria preciso escrever a ponte de validação à mão em cada formulário.
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
- **Autorização**: por **recurso**, não por papel da conta (a conta não tem papel fixo — ver
  R19). Dependências FastAPI: `current_user`; `require_artist` (conta tem perfil de
  tatuador); `studio_access(studio_id, level)` com `level` = `member` (dono ou membro) ou
  `owner`. Checagens de posse no serviço (ex.: só o tatuador do agendamento aprova).
- **Rationale**: sessão opaca é revogável (logout, troca de senha), evita armazenar JWT no
  browser e é simples com mesmo domínio.
- **Alternatives considered**: JWT stateless (revogação difícil, sem benefício com um único
  backend); FastAPI-Users (traz mais do que o necessário e acopla o modelo de usuário).

## R4. Concorrência: pedido sem reserva, regras na aprovação

Desde a clarificação de 2026-11-13, o **pedido não reserva vaga**: o cliente informa data e
faixa preferidas, e o intervalo só passa a existir quando o tatuador aprova (FR-008b,
FR-011a). Por isso as regras de ocupação ficam todas na **aprovação**.

- **Decision**:
  1. **Sobreposição do tatuador (FR-010)**: constraint de exclusão no PostgreSQL com
     `btree_gist`: `EXCLUDE USING gist (artist_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
     WHERE (status = 'confirmed')`. Pendentes não têm intervalo (`starts_at`/`ends_at` nulos)
     e ficam fora da regra.
  2. **Aprovação (FR-007a, FR-007c, FR-008b)**: transação com `pg_advisory_xact_lock` da
     chave do **estúdio** (serializa aprovações do mesmo estúdio). Dentro do lock: (a) o
     intervalo está inteiramente coberto pelas faixas da data (união de faixas contíguas);
     (b) para cada faixa tocada, o pico de sessões confirmadas sobrepostas dentro dela + 1 ≤
     capacidade da faixa; (c) o pico de sessões confirmadas do estúdio no intervalo + 1 ≤
     `seats`; (d) dia da semana em `affiliation.weekdays` e início no futuro. Picos por
     varredura dos limites dos intervalos (sweep line). Depois, `UPDATE` para `confirmed`; a
     constraint de exclusão garante (FR-010) mesmo entre estúdios diferentes.
  3. **Limite de pendentes por cliente (FR-011c)**: e-mail e telefone normalizados em colunas
     próprias; na **criação** e na **remarcação**, `pg_advisory_xact_lock` das chaves do
     e-mail e do telefone (ordenadas, para evitar deadlock) e checagem `< 2` pendentes com o
     mesmo e-mail **ou** telefone. A criação não trava o estúdio (não há ocupação a checar).
  4. Violação → `409` com código específico: `slot_unavailable` (fora das faixas, dia ou
     janela), `slot_full` (capacidade ou macas), `artist_unavailable` (sobreposição),
     `pending_limit_reached`. Alterar `seats` ou a capacidade de uma faixa abaixo do pico de
     sessões confirmadas futuras usa o mesmo lock do estúdio → `409 seats_below_bookings` /
     `slot_has_bookings`.
  5. **Remarcação (FR-011d)**: só troca a preferência (data e faixa), volta a `pending` e
     zera `starts_at`/`ends_at` (liberando o intervalo, se confirmado), numa transação com os
     locks do cliente. Não há disputa de vaga, logo não falha por ocupação.
- **Rationale**: atende SC-003 com as regras checadas sob lock no único momento em que um
  intervalo passa a ocupar maca. A trava por estúdio cobre faixas sobrepostas com inícios
  diferentes; a contenção é baixa (aprovações de um mesmo estúdio no mesmo instante são raras).
- **Alternatives considered**: reservar uma duração padrão no pedido (rejeitado na
  clarificação: o tatuador define a duração); isolamento SERIALIZABLE em tudo (retries
  frequentes).

## R5. Faixas de atendimento: padrão semanal + exceções (FR-007, FR-007b)

- **Decision**: faixas **calculadas sob demanda**, não materializadas:
  - `weekly_rule` (dia da semana, hora início/fim no fuso do estúdio, capacidade);
  - `date_override` (data específica: bloqueada, ou faixas substitutas);
  - `one_off_slot` (faixa avulsa com início/fim/capacidade).
  Dois cálculos usam essas faixas:
  - **Datas oferecidas ao cliente** (`GET /studios/{slug}/availability`): para cada data na
    janela de antecedência (R21) e num dia de atendimento do tatuador, lista as faixas da data
    que ainda têm **ao menos 30 min livres** para aquele tatuador (capacidade, macas e agenda
    confirmada dele em qualquer estúdio). Datas sem nenhuma faixa assim não aparecem.
  - **Intervalos livres para a aprovação** (`GET /bookings/{id}/free-intervals`): para a data
    pedida, a união das faixas recortada pelas ocupações (capacidade, macas, sessões do
    tatuador), em blocos de 30 min. É uma ajuda de tela; a validação definitiva é a do R4.
  O agendamento confirmado guarda o próprio intervalo, então mudanças nas faixas não afetam
  sessões confirmadas. Pedidos pendentes para uma data que ficou sem faixas simplesmente não
  podem ser aprovados.
- **Rationale**: atende FR-007b sem job de geração nem limpeza de linhas futuras.
- **Alternatives considered**: materializar faixas por N semanas (job recorrente e
  reconciliação ao editar o padrão).
- **Fuso horário**: `timezone` IANA por estúdio; faixas em hora local, conversão com
  `zoneinfo`; armazenamento em UTC (`timestamptz`); `requested_date` é data local do estúdio.

## R6. Mensageria: outbox transacional + RabbitMQ + Celery

- **Decision**: processamento assíncrono com **Celery** (workers e Beat) sobre **RabbitMQ**,
  alimentado por um **outbox transacional** no PostgreSQL:
  1. **Produtor (API)**: o caso de uso grava a mudança de negócio, a `notification` (in-app +
     dados do e-mail) e uma linha em `outbox_message` (`task` + `payload` JSON) **na mesma
     transação** do SQLAlchemy. Nunca publica direto no broker: ou tudo é commitado, ou
     nada é. Um único serviço `Outbox.enqueue(session, message)` monta a linha; os casos
     de uso não conhecem o Celery.
  2. **Relay (`python -m app.outbox_relay`)**: processo dedicado que, a cada 1 s, lê até 100
     mensagens com `published_at IS NULL` usando `SELECT ... FOR UPDATE SKIP LOCKED`,
     publica cada uma no RabbitMQ (`send_task` do Celery com **publisher confirms**,
     `confirm_publish=True`) e marca `published_at` na mesma transação da leitura. Falha
     ao publicar → rollback, a linha continua pendente e é retentada. `SKIP LOCKED`
     permite mais de uma réplica do relay sem duplicar envio.
  3. **Consumidores (Celery workers)**: tasks por módulo (ex.:
     `notifications.send_email`). Entrega **at-least-once**: `acks_late=True`,
     `task_reject_on_worker_lost=True`, filas duráveis e mensagens persistentes. Toda task é
     **idempotente**: `send_email` relê a `notification` com `FOR UPDATE SKIP LOCKED` e
     ignora se `email_sent_at` já está preenchido. Falhas de SMTP usam `autoretry_for` com
     **backoff exponencial** (30 s → ~30 min, com jitter), até 6 tentativas; esgotadas, a
     mensagem vai para uma **dead-letter queue** (`email.dlq`) e a notificação fica com
     `email_failed_at`, visível nos logs/alerta.
  4. **Tarefas periódicas (Celery Beat, instância única)**: `expire_pending_bookings` e
     `expire_affiliation_invites` (1 min), `complete_past_bookings` (5 min),
     `send_client_reminders` (5 min, R22), `send_artist_daily_digest` (18:00
     `America/Sao_Paulo`, R22), `anonymize_old_bookings` e `purge_audit_log` (1 h). As
     periódicas fazem a mudança de negócio e as notificações numa transação e, como as
     demais, **enfileiram e-mails pelo outbox** em vez de enviá-los.
  `expires_at = min(requested_at + 48h, início da faixa pedida)` é calculado na criação e
  na remarcação (FR-011b, R15). A aprovação também recusa (`invalid_transition`) um pedido
  com `expires_at <= now` ainda não processado pela task, sem depender do Beat.
- **Filas**: `email` (envio), `default` (demais tasks), `email.dlq` (dead-letter, via
  `x-dead-letter-exchange` do RabbitMQ). Prefetch 1 nos workers de e-mail (tarefas lentas de
  I/O).
- **Celery com código assíncrono**: tasks do Celery são síncronas; o handler de cada task
  roda o caso de uso assíncrono existente com `asyncio.run(...)`, usando um engine
  SQLAlchemy com `NullPool` no worker (evita pool preso a um event loop). Assim os mesmos
  repositórios e serviços servem à API e aos workers, sem uma segunda camada síncrona
  (Princípio I, sem duplicação).
- **Rationale**: o outbox garante atomicidade entre o dado e a mensagem (sem "salvou mas não
  publicou" nem "publicou mas deu rollback"), e o RabbitMQ + Celery trazem retentativa com
  backoff, dead-letter, escala horizontal de consumidores e agendamento de periódicas sem
  escrever essa infraestrutura à mão.
- **Alternatives considered**: worker caseiro com loop `asyncio` lendo o Postgres (plano
  anterior; trocado por decisão do usuário em 2026-11-13 em favor de uma mensageria
  padrão); publicar no RabbitMQ direto da requisição (perde a atomicidade com o banco);
  Redis como broker (sem dead-letter nativo e sem garantias de persistência equivalentes);
  `BackgroundTasks` do FastAPI (perde mensagens em restart).
- **Testes**: o relay publica por um `Protocol` `TaskPublisher` (produção: Celery; testes:
  fake em memória). Testes de integração verificam que a linha de outbox nasce/morre com a
  transação de negócio e chamam os handlers das tasks diretamente; um teste de fumaça no CI
  sobe RabbitMQ e um worker real para um envio ponta a ponta até o Mailpit.
- **E-mail**: envio pela API do Gmail em produção e por SMTP para o Mailpit em
  desenvolvimento, atrás do mesmo `Mailer` (R23).

## R7. Imagens do portfólio com MinIO

- **Decision**: upload multipart pela API (`POST /api/v1/me/artist/portfolio`), limite 10 MB,
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
  Estúdio por estilo (FR-017): `EXISTS` sobre `affiliation` ativa + `artist_style` do
  tatuador, sem coluna de estilo no estúdio (sempre coerente com os vínculos).
- **Rationale**: volume da v1 não justifica motor de busca dedicado.
- **Alternatives considered**: Elasticsearch/Meilisearch (infra extra, YAGNI).

## R10. Link de acesso do visitante (FR-002)

- **Decision**: ao criar um agendamento como visitante, gera token aleatório de 32 bytes;
  só o hash SHA-256 é salvo no agendamento. O e-mail traz
  `/agendamentos/acesso/{token}`, que permite ver, remarcar e cancelar aquele agendamento.
  Quando **qualquer conta** verifica o e-mail, agendamentos de visitante com o mesmo e-mail
  são associados a ela (FR-002a).
- **Rationale**: o token vazado não compromete outros agendamentos; associar só após
  verificação do e-mail impede que alguém "tome" agendamentos de outra pessoa.
- **Consequência**: a associação só acontece após a verificação do e-mail da conta.

## R11. Qualidade e testes (Constituição)

- **Decision**:
  - Backend: `ruff` (lint + format), `mypy --strict` nos módulos de domínio, `pytest` +
    `pytest-asyncio` + `httpx.AsyncClient`; testes de integração contra PostgreSQL real
    (container do compose de teste).
  - Frontend: ESLint + Prettier, Vitest + Testing Library; Playwright para e2e em dois
    viewports (mobile 390×844 e desktop 1440×900). `@axe-core/playwright` roda checagens
    automáticas de acessibilidade (contraste, labels) nos mesmos testes e2e, cobrindo o
    mínimo definido em contracts/ui-routes.md sem auditoria manual a cada mudança.
  - Repositório: `pre-commit` com hooks de ruff, eslint/prettier e
    `conventional-pre-commit` (validação de commits Angular); GitHub Actions rodando lint,
    testes e e2e.
- **Rationale**: cobre os itens obrigatórios de "Padrões de Qualidade" (lint/format no
  pre-commit e CI, validação de mensagem de commit, testes de regra de negócio).
- **Abstrações (Princípios III e V)**: `Protocol` apenas onde há duas implementações reais:
  `Clock`, `ObjectStorage` e `Mailer` (produção + fake de teste) e os providers entre
  módulos (`BookedCountProvider`, `ArtistBusyProvider`). Repositórios são classes concretas
  testadas contra PostgreSQL real; não há fakes de repositório.
- **Assinaturas (Princípio IV, máx. 3 parâmetros)**: funções com mais entradas recebem um
  objeto de parâmetros, por exemplo `generate_occurrences(schedule: ScheduleSnapshot,
  period: DateRange, now)` e `notify(session, request: NotificationRequest)`.

## R12. Ambiente e deploy

- **Decision**: Docker Compose com serviços `nginx`, `frontend` (build → estático servido
  pelo nginx), `backend` (API), `outbox-relay`, `celery-worker`, `celery-beat` (réplica
  única), `rabbitmq` (3.13, com plugin de management), `postgres` (16), `minio`, `mailpit`
  (apenas dev). `backend`, `outbox-relay`, `celery-worker` e `celery-beat` usam a mesma
  imagem com comandos diferentes.
  Configuração por variáveis de ambiente; segredos fora do repositório (`.env` no
  `.gitignore`, `.env.example` versionado).
- **Rationale**: mesmo ambiente para dev, CI e um servidor único de produção na v1.

## R13. Recuperação de senha (FR-003a)

- **Decision**: tabela `password_reset_token` (hash SHA-256 do token de 32 bytes, expira em
  1h, `used_at`). `POST /auth/password-reset/request` sempre responde `204`, cadastrado ou
  não, e só envia e-mail (pelo outbox) se a conta existir. `POST /auth/password-reset/confirm`
  troca a senha, marca o token como usado, invalida os demais tokens da conta e **apaga todas
  as sessões** do usuário. nginx limita a taxa da rota de solicitação.
- **Rationale**: resposta uniforme impede descobrir quais e-mails têm conta; token de uso
  único e curto reduz a janela de abuso; reaproveita outbox e padrão de token do link de
  visitante (R10).
- **Alternatives considered**: código numérico por e-mail (mais sujeito a força bruta);
  magic link de login (muda o modelo de autenticação definido na spec).

## R14. Privacidade: anonimização e exclusão de conta (FR-018 a FR-020)

- **Decision**:
  - **Visibilidade (FR-018)**: schemas de resposta separados; contato do cliente só em
    schemas usados por rotas autorizadas às partes do agendamento. Teste de integração
    varre as rotas públicas garantindo ausência de e-mail/telefone.
  - **Anonimização (FR-019)**: task periódica `anonymize_old_bookings` no Celery Beat (uma vez por hora, R6)
    substitui `client_name` por "Cliente anonimizado", zera e-mail/telefone/notas e o
    `guest_token_hash` de agendamentos encerrados há 12 meses: `ends_at <= now() − 12
    meses` para os que tiveram sessão, ou fim da `requested_date` `<= now() − 12 meses` para
    os que nunca foram confirmados (recusados, expirados, cancelados pendentes), marcando
    `anonymized_at`. Notificações desses agendamentos têm `recipient_email` anonimizado.
  - **Exclusão de estúdio (FR-020a)**: `DELETE /studios/{id}` (dono, confirmação de senha):
    cancela agendamentos futuros ativos do estúdio (`cancelled_by = studio`), encerra
    vínculos e convites, remove os membros, marca `studio.deleted_at`; nome é mantido no
    histórico das outras partes.
  - **Exclusão de conta (FR-020)**: `DELETE /me` com confirmação de senha, numa transação,
    compondo os casos de uso existentes: exclui o estúdio de que é dono (acima), remove a
    conta das equipes, exclui o perfil de tatuador (cancela os agendamentos dele com
    `cancelled_by = artist`, encerra vínculos, remove portfólio e fotos do MinIO após o
    commit), cancela os agendamentos em que é cliente (`cancelled_by = client`), apaga
    sessões e tokens e **anonimiza a conta** (soft delete: `deleted_at`, e-mail trocado por
    valor único inválido, hash de senha removido). Tatuador mantém o `display_name` no
    histórico das outras partes; nome e contato de cliente são apagados também nos
    agendamentos passados.
- **Rationale**: soft delete preserva a integridade referencial do histórico de
  agendamentos dos outros usuários; anonimizar em vez de apagar cumpre a LGPD mantendo dados
  não pessoais.
- **Alternatives considered**: `DELETE` em cascata (apagaria o histórico das outras partes);
  exclusão só via suporte (rejeitada na clarificação).

## R15. Expiração, cancelamento e falta (FR-011, FR-011b, FR-011e)

- **Expiração**: `expires_at = min(requested_at + 48h, início da faixa pedida)`, com o início
  da faixa calculado em `requested_date` + `requested_window_start` no fuso do estúdio.
  Funciona para pedidos do mesmo dia (antecedência mínima 0h): um pedido às 10h para a faixa
  14:00–18:00 de hoje expira às 14:00. Substitui a regra provisória "00h da data pedida"
  (clarificação de 2026-11-13).
- **Decision**: `cancel` de `pending` é sempre permitido enquanto pendente; de `confirmed`
  exige `now < starts_at`, senão `409 booking_started`. Vale para cliente, tatuador, estúdio
  e link de visitante.
- **Falta**: `mark_no_show` (tatuador do agendamento ou dono/membro do estúdio) exige
  `confirmed` e `starts_at <= now < starts_at + 7 dias` (`409 no_show_window_closed` fora
  disso); `unmark_no_show` volta a `confirmed` na mesma janela. A task periódica
  `complete_past_bookings` passa para `completed` as sessões `confirmed` com
  `starts_at + 7 dias <= now`. Marcações do lado do estúdio são auditadas.

## R16. Detalhes do pedido e imagens de referência (FR-008a)

- **Decision**:
  - `POST /bookings` passa a ser **multipart**: campos do pedido + até 3 arquivos
    `references`. Descrição (1–1000 caracteres), local do corpo (enum fixo) e tamanho
    (`P`/`M`/`G`) são obrigatórios.
  - As imagens passam pelo mesmo processamento do portfólio (R7: Pillow, tipo real, ≤ 10 MB,
    EXIF removido), mas geram **uma** variante WebP (largura máx. 1600 px). O processamento
    acontece **antes** de abrir a transação e dos locks do cliente (R4), para não segurar
    locks durante trabalho de CPU.
  - Ficam num bucket **privado** `booking-refs` (sem política anônima, não exposto pelo
    nginx), na chave `booking-refs/{booking_id}/{n}.webp`. O upload ao storage ocorre dentro
    do fluxo de criação antes do commit; se a transação falhar, o prefixo é apagado no
    tratamento de erro.
  - Leitura por `GET /bookings/{id}/references/{n}` (partes do agendamento) e
    `GET /guest/bookings/{token}/references/{n}` (visitante): o backend faz streaming do
    objeto com `Cache-Control: private, no-store`.
  - Anonimização (FR-019) e exclusão de conta de cliente (FR-020) apagam o prefixo do
    agendamento e as linhas de referência.
- **Rationale**: as imagens são dados do cliente, visíveis só às partes (FR-008a/FR-018), e
  por isso não podem usar o bucket público. Streaming pelo backend dispensa expor o MinIO ou
  gerar URLs assinadas com host interno; o volume é baixo (≤ 3 imagens por pedido, vistas
  poucas vezes).
- **Alternatives considered**: URL pré-assinada de GET (exigiria expor o MinIO por outro
  caminho do nginx com repasse de assinatura); salvar imagens no Postgres (incha backups e
  consultas).
- **Moderação**: fora do escopo da v1 (spec, Assumptions). O risco fica contido porque as
  imagens nunca são públicas.

## R17. Verificação de e-mail como condição de visibilidade (FR-001a)

- **Decision**: o cadastro emite token de verificação (R13: mesmo formato, validade 48h)
  para toda conta; `POST /auth/verify-email/resend` reenvia (limitado no nginx). Um
  filtro único de repositório, "publicável", é aplicado: tatuador publicável = conta com
  `email_verified_at IS NOT NULL AND deleted_at IS NULL`; estúdio publicável = estúdio sem
  `deleted_at` cujo **dono** é publicável. Ele vale em páginas públicas, buscas, lista de tatuadores do
  estúdio e disponibilidade. A criação de agendamento exige estúdio e tatuador publicáveis
  (`409 slot_unavailable` caso contrário, sem revelar o motivo ao visitante). Estúdio e
  tatuador não verificados usam o painel normalmente, com banner de aviso.
- **Rationale**: garante que notificações de pedidos cheguem ao destinatário (FR-012); um
  filtro centralizado evita esquecer a regra em alguma consulta.

## R18. Convites de vínculo (FR-004a)

- **Decision**: `affiliation.expires_at = invited_at + 24h`; a task periódica do Celery Beat expira (R6) e notifica
  o estúdio (`affiliation_expired`). O estúdio cancela com `POST /affiliations/{id}/cancel`
  (`invited` → `cancelled`, sem notificação ao tatuador; o convite some da lista dele).
  Aceitar/recusar convite vencido ou cancelado → `409 invitation_not_valid`; aceitar checa
  `expires_at` também na hora, sem depender da task periódica. O índice único parcial (só `invited`
  e `active`) permite convidar de novo depois.

## R19. Modelo de contas: pessoa com perfis e papéis por estúdio (FR-001, FR-003)

- **Decision**: `user` representa a pessoa (nome, e-mail, telefone, senha) e **não tem
  papel**. Capacidades são relações:
  - `artist_profile` 0..1 por `user` (perfil de tatuador);
  - `studio` é entidade própria com `owner_user_id` **único** (no máximo 1 estúdio por dono,
    garantido por índice único parcial `WHERE deleted_at IS NULL`);
  - `studio_member(studio_id, user_id)` N:N, com restrição de que o dono não é membro do
    próprio estúdio;
  - agendar como cliente não exige relação: qualquer `user` (ou visitante).
  O frontend recebe em `GET /auth/me` as "áreas" da conta (tatuador? estúdio próprio? estúdios
  em que é membro) e mostra um seletor de área no painel; as rotas do estúdio são
  parametrizadas por `studio_id` (`/studios/{id}/...`), não por `/me`.
- **Rationale**: atende "o dono pode ser tatuador" e "tatuador pode ser membro de vários
  estúdios" sem contas duplicadas. Autorização por recurso (R3) é a forma natural quando uma
  mesma pessoa tem permissões diferentes em estúdios diferentes.
- **Alternatives considered**: papel único na conta (modelo anterior, rejeitado na
  clarificação); RBAC genérico com tabela de permissões (YAGNI: só existem dois papéis, dono e
  membro).
- **Notificações do estúdio (FR-024)**: notificação in-app ganha `studio_id` opcional; as do
  estúdio são listadas para dono e membros pela relação, sem duplicar linhas. E-mails do
  estúdio vão só para o dono.

## R20. Equipe e registro de ações (FR-021 a FR-023)

- **Decision**:
  - **Convite de membro**: `studio_member_invite` (e-mail convidado, token hash, expira em 7
    dias). Aceitar exige estar logado com uma conta **cujo e-mail é o convidado**; sem conta,
    o link leva ao cadastro com o e-mail pré-preenchido e aceita em seguida. Remover membro
    apaga a linha de `studio_member`; como a autorização é checada a cada requisição, o
    acesso cai na hora, sem mexer nas sessões da pessoa.
  - **Registro de ações**: tabela `studio_audit_log` (append-only) com `studio_id`,
    `actor_user_id`, `actor_name` (snapshot), `action` (enum), `target` (tipo e id),
    `changes` (jsonb com antes/depois resumido) e `created_at`. É gravado **na mesma
    transação** da ação, por um decorator/serviço `AuditTrail.record(session, entry)`
    chamado pelos casos de uso de gestão — nunca pelo router. A task periódica `purge_audit_log` (R6) apaga entradas com
    mais de 12 meses.
  - Ações registradas: regras, exceções, avulsos, macas, janela de antecedência, dias de
    atendimento, convites/vínculos, membros, cancelamentos de agendamento pelo estúdio.
- **Rationale**: gravar na mesma transação garante que não exista mudança sem registro;
  centralizar num serviço evita que cada caso de uso monte a entrada de um jeito.
- **Alternatives considered**: triggers no banco (perdem o autor da requisição sem gambiarra
  de variável de sessão); log em arquivo (não consultável pelo dono).

## R21. Dias de atendimento e janela de antecedência (FR-007d, FR-007e)

- **Decision**: `affiliation.weekdays` (smallint bitmask, 7 bits, padrão 127 = todos) e
  `studio.min_notice_hours` (0–72, padrão 24) e `studio.max_advance_days` (7–180, padrão
  60). As datas oferecidas são filtradas pelo dia da semana (local do estúdio) e pela janela:
  uma data entra se alguma faixa dela começa em `[now + min_notice, now + max_advance]`.
  Criação e remarcação revalidam a data pedida; a aprovação revalida o dia da semana e exige
  início no futuro (código `slot_unavailable`). Alterações não tocam agendamentos existentes.
- **Rationale**: bitmask é compacto, fácil de indexar/validar e de exibir como 7 checkboxes.

## R22. Lembretes (FR-012a)

- **Decision**: duas tasks periódicas no Celery Beat (R6), que enfileiram os e-mails pelo
  outbox:
  - `send_client_reminders`: agendamentos `confirmed` com `starts_at` entre `now + 23h` e
    `now + 24h`, `reminder_sent_at IS NULL` e `decided_at <= starts_at − 24h`; cria a notificação, a
    mensagem de outbox e marca `reminder_sent_at` na mesma transação. Remarcação zera
    `reminder_sent_at`.
  - `send_artist_daily_digest`: agendada no Beat com `crontab(hour=18, minute=0)` no fuso
    `America/Sao_Paulo`; agrupa por tatuador os
    agendamentos `confirmed` do dia seguinte (dia de Brasília) e grava uma notificação por
    tatuador com agendamentos. Uma tabela `job_run(job, run_date)` com chave única garante
    execução única por dia mesmo com restart.
- **Rationale**: marcas de envio na própria linha/tabela tornam as tasks idempotentes mesmo
  com reentrega do RabbitMQ ou restart do Beat; a janela de 1h dos lembretes tolera atrasos
  de execução.

## R23. E-mail pela API do Gmail (conta pessoal no MVP)

- **Decision**: `Mailer` ganha duas implementações: `GmailApiMailer` (produção) e
  `SmtpMailer` (desenvolvimento e testes, apontando para o Mailpit). O `GmailApiMailer`
  monta a mensagem MIME (stdlib `email`) e chama
  `POST https://gmail.googleapis.com/gmail/v1/users/me/messages/send` com `httpx`, usando um
  access token obtido do **refresh token** OAuth da conta @gmail.com pelo `google-auth`.
  - **Autorização**: app OAuth no Google Cloud com escopos `gmail.send` e `gmail.modify`,
    em status **"In production"** (em "Testing" o refresh token expira em 7 dias). Para uso
    só pela conta do próprio projeto, o app fica não verificado e o consentimento único
    aceita o aviso do Google. O refresh token é obtido uma vez pelo script
    `python -m app.tools.gmail_authorize` e guardado como segredo (`GMAIL_REFRESH_TOKEN`),
    nunca versionado.
  - **Cota (cerca de 500 destinatários/dia em conta pessoal)**: erros de cota
    (`rateLimitExceeded`, `dailyLimitExceeded`, HTTP 429) não contam como falha: a task se
    reagenda com `countdown` crescente até 1 h, sem consumir as 6 tentativas de falha real.
    A ordem aproximada de envio é a de criação (fila FIFO do RabbitMQ). Um contador diário
    exposto no `/health` (`mail_sent_today`) mostra a proximidade do limite.
  - **Falha de autorização** (`invalid_grant`): a task não re-tenta em loop; registra erro,
    marca `/health` com `mail: "error"` e as mensagens ficam na fila até o token ser
    renovado.
- **Detecção de devoluções (FR-002b)**: cada e-mail sai com um cabeçalho `Message-ID`
  gerado pelo sistema e guardado em `notification.email_message_id`. A task periódica
  `notifications.scan_bounces` (a cada 10 min) consulta a caixa com
  `from:mailer-daemon@googlemail.com newer_than:3d -label:tatuagem-processado`, extrai o
  `Message-ID` original do relatório de entrega (DSN), marca `email_failed_at` e
  `delivery_failure_reason` na notificação e aplica o rótulo `tatuagem-processado`. Uma
  notificação de cliente com falha faz o agendamento exibir o aviso "e-mail do cliente não
  entregue" ao estúdio e ao tatuador.
- **Rationale**: atende a decisão do usuário (Gmail no MVP) com uma dependência pequena
  (`google-auth` só para renovar tokens; `httpx` já é usado nos testes e passa a ser
  dependência de produção) em vez do SDK completo `google-api-python-client`. Manter o
  `SmtpMailer` permite rodar tudo localmente sem conta Google.
- **Alternatives considered**: SMTP do Gmail com senha de app (exige verificação em duas
  etapas e não dá acesso estruturado à caixa para ler devoluções); provedor transacional
  (SES, Postmark, Resend: adiado para depois do MVP, junto com domínio próprio).
- **Riscos aceitos**: limite diário baixo; termos do Google não preveem envio automatizado
  de produto por conta pessoal (risco de bloqueio da conta); remetente @gmail.com.

## R24. Recuperação do link do visitante (FR-002b)

- **Decision**: a resposta de `POST /bookings` para visitante inclui `guest_access_url`
  (exibido só nessa resposta, com botão de copiar na tela de confirmação). Os tokens de
  acesso continuam guardados só como hash; para reenviar, o sistema **gera um novo token**
  para cada agendamento ativo (`pending` ou `confirmed`) daquele e-mail, invalidando o
  anterior, e manda um único e-mail com todos os links (`POST /guest/resend-links`,
  resposta sempre `204`; limitado no nginx). O e-mail de criação do pedido
  (`booking_received`) também carrega o link.
- **Rationale**: o link nunca precisa ser armazenado em texto aberto; quem tem acesso ao
  e-mail recupera o acesso, e mais ninguém.
