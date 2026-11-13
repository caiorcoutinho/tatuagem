# Quickstart: validação ponta a ponta

**Feature**: `001-agendamento-tatuagem` · Referências: [API](./contracts/api.md),
[UI](./contracts/ui-routes.md), [modelo de dados](./data-model.md)

Este guia mostra como subir o ambiente e validar que a feature funciona. Ele não descreve
implementação.

## Pré-requisitos

- Docker + Docker Compose v2
- Node 22 LTS e Python 3.12 (só para rodar lint/testes fora dos containers)
- `cp .env.example .env` (valores de dev já preenchidos; nenhum segredo real)

## Subir o ambiente

```bash
docker compose up -d --build          # nginx, backend, outbox-relay, celery-worker, celery-beat,
                                      # rabbitmq, postgres, minio, mailpit
docker compose exec backend alembic upgrade head
docker compose exec backend python -m app.seed   # estilos de tatuagem + dados de demonstração
```

| Serviço | URL |
|---|---|
| Aplicação (nginx) | http://localhost:8080 |
| API / OpenAPI | http://localhost:8080/api/v1/docs |
| Mailpit (e-mails) | http://localhost:8025 |
| Console MinIO | http://localhost:9001 |
| RabbitMQ management (filas `email`, `default`, `email.dlq`) | http://localhost:15672 |

Esperado: `GET http://localhost:8080/api/v1/health` → `{"status":"ok","db":"ok","storage":"ok","broker":"ok","outbox_backlog":0}`.

## Testes automatizados

```bash
docker compose -f compose.test.yaml run --rm backend-tests   # pytest (unit + integração com Postgres real)
cd frontend && npm ci && npm test                            # Vitest
cd frontend && npx playwright test                           # e2e em mobile (390×844) e desktop (1440×900)
pre-commit run --all-files                                   # ruff, eslint, prettier
```

Esperado: tudo verde. Os testes de integração cobrem os cenários marcados com 🧪 abaixo.

## Personagens

Crie as contas pela UI (`/cadastro`) e confirme cada e-mail pelo link no Mailpit:

| Conta | Papel nos cenários |
|---|---|
| **Ana** | cria o **Estúdio A** (fuso `America/Sao_Paulo`, 2 macas) e **também** o perfil de tatuadora "Ana Tattoo" |
| **Bruno** | perfil de tatuador **X**; depois vira membro do Estúdio A |
| **Carla** | perfil de tatuadora **Y** |
| **Davi** | cria o **Estúdio B**; também tem perfil de tatuador **Z** |
| visitante | sem conta, usa uma aba anônima |

## Cenários de validação manual

Use a UI em http://localhost:8080 no desktop e no modo responsivo do navegador (iPhone 12).

### V1: Estúdio, vínculo, dias e horários (US1) 🧪
1. Como Ana, na área do Estúdio A → `tatuadores`, convide X (Bruno) e "Ana Tattoo".
   **Esperado**: convites com 24h restantes; ninguém aparece ainda em `/estudios/estudio-a`.
2. Bruno aceita em `/painel/tatuador/estudios`; Ana aceita o próprio convite na área
   Tatuador. **Esperado**: os dois aparecem no perfil público do estúdio.
3. Em `horarios`, crie o padrão seg–sex 10:00–14:00, capacidade 2, e mantenha a janela
   padrão (24h a 60 dias). **Esperado**: ao agendar com X, aparecem os dias úteis a partir de
   amanhã + 24h, até 60 dias.
4. Em `tatuadores`, deixe X só às segundas e quartas (FR-007d). **Esperado**: com X só
   aparecem segundas e quartas; com "Ana Tattoo", todos os dias úteis. Bruno vê os dias em
   `/painel/tatuador/estudios` sem poder editar.
5. Bloqueie a próxima segunda. **Esperado**: a data some da disponibilidade.
6. Mude a antecedência mínima para 0h e a máxima para 7 dias. **Esperado**: horários de hoje
   (ainda não iniciados) aparecem; nada além de 7 dias.

### V2: Agendamento como visitante e aprovação (US2) 🧪
1. Em aba anônima, abra `/tatuadores/tatuador-x` → "Agendar". **Esperado**: é preciso
   escolher o estúdio antes de ver horários.
2. Escolha um horário, preencha descrição, local do corpo e tamanho, anexe 2 imagens e
   informe nome, e-mail e telefone. **Esperado**: status "pendente"; e-mail com link
   `/agendamentos/acesso/...`; X vê os detalhes e as imagens em
   `/painel/tatuador/pedidos`. A URL de uma imagem aberta sem sessão → `401`/`404`.
3. X aprova. **Esperado**: `confirmed`; cliente e Ana (e-mail do dono) recebem e-mail; a
   notificação in-app aparece na área do Estúdio A.
4. Pelo link do e-mail, cancele. **Esperado**: `cancelled`; vaga volta; X e estúdio notificados.

### V3: Capacidade, macas e concorrência (FR-007a, FR-007c, FR-010, SC-003) 🧪
1. Vincule também Y (Carla) ao Estúdio A. Agende X e "Ana Tattoo" no mesmo horário.
   **Esperado**: o horário some para Y (`slot_full`).
2. Vincule X também ao Estúdio B (de Davi) com horário sobreposto. **Esperado**: em B, X não
   aparece livre naquele intervalo; forçar via API → `409 artist_unavailable`.
3. Crie um avulso sobreposto a um horário do padrão no Estúdio A e ocupe as 2 macas com
   horários de inícios diferentes. **Esperado**: ambos somem para Y; via API `409 slot_full`.
4. Ana tenta reduzir as macas para 1. **Esperado**: `409 seats_below_bookings`.
5. Teste automatizado: 20 requisições simultâneas para a última vaga, incluindo horários
   sobrepostos com inícios diferentes. **Esperado**: exatamente 1 `201`.

### V4: Expiração, conclusão e lembretes (FR-011b, FR-012a) 🧪
Com relógio controlado nos testes de integração das tasks periódicas (ou ajustando datas no
banco de dev e disparando a task pelo Beat):
1. Pedido sem resposta por 48h. **Esperado**: `expired`, vaga liberada, cliente notificado.
2. Agendamento confirmado que começa em 24h. **Esperado**: um único lembrete por e-mail ao
   cliente, com links de ver/remarcar/cancelar; nenhum para um confirmado faltando 3h.
3. Relógio às 18:00 (Brasília) com X tendo 2 agendamentos confirmados amanhã, em A e B.
   **Esperado**: um único e-mail de resumo para Bruno listando os dois, cada um no fuso do
   estúdio; reiniciar o `celery-beat` ou o `celery-worker` não reenvia. Carla, sem agendamentos amanhã, não recebe nada.
4. Após `ends_at`, a task periódica marca `completed`.

### V4b: Mensageria e outbox (research.md R6) 🧪
1. Teste de integração: uma ação que gera e-mail (ex.: aprovar pedido) cria a
   `notification` e a `outbox_message`; forçar rollback da transação não deixa nenhuma das
   duas.
2. Pare o RabbitMQ (`docker compose stop rabbitmq`) e aprove um pedido. **Esperado**: a
   aprovação funciona; a `outbox_message` fica com `published_at` nulo. Suba o RabbitMQ:
   em segundos o relay publica e o e-mail chega ao Mailpit, uma única vez.
3. Pare o Mailpit e aprove outro pedido. **Esperado**: a task re-tenta com backoff (visível
   nos logs); após esgotar, a mensagem aparece em `email.dlq` e a notificação fica com
   `email_failed_at`.
4. Reentrega: reenvie manualmente a mesma mensagem pela UI do RabbitMQ. **Esperado**: a task
   reconhece `email_sent_at` preenchido e não envia de novo.

### V5: Desvínculo, edição e cancelamento (edge cases) 🧪
1. Com X confirmado no Estúdio A, encerre o vínculo. **Esperado**: agendamento mantido;
   novo agendamento → `409 affiliation_inactive`.
2. Exclua um avulso com agendamento ativo. **Esperado**: `409 slot_has_bookings`.
3. Remova a quarta dos dias de X com um agendamento numa quarta. **Esperado**: permitido;
   o agendamento continua; novas quartas somem para X.
4. Cancelar um agendamento já iniciado. **Esperado**: `409 booking_started`; botão oculto.

### V6: Conta que agenda como cliente (FR-002a) 🧪
1. Crie a conta **Eva** com o mesmo e-mail usado pelo visitante em V2 e verifique o e-mail.
   **Esperado**: o agendamento de visitante aparece em `/painel/agendamentos`.
2. Bruno (tatuador X) agenda uma tatuagem com Y pela própria conta. **Esperado**: aceito; o
   pedido aparece em "Meus agendamentos" dele. Bruno tenta agendar com X → `409 self_booking`.

### V7: Portfólio (US3, SC-004)
1. Como X, envie 3 imagens (JPEG, PNG, WebP) com título e estilo. **Esperado**: aparecem na
   grade de `/tatuadores/tatuador-x`; abrir um trabalho mostra a imagem ampliada e "Agendar".
2. Envie um `.txt` renomeado para `.jpg`. **Esperado**: `400 invalid_image`.
3. Com o seed de 100 trabalhos (`python -m app.seed --portfolio 100`), meça no DevTools
   (rede "Fast 4G"): a grade fica visível em < 2 s.

### V8: Agenda consolidada e busca (US4, US5)
1. Com X agendado em A e B, abra `/painel/tatuador/agenda` como Bruno. **Esperado**: os dois
   agendamentos, cada um com o nome do estúdio; cancelar disponível até o início.
2. Em `/`, filtre tatuadores por estilo e cidade. **Esperado**: só correspondentes.
3. Filtre estúdios pelo estilo de X. **Esperado**: A e B aparecem; encerrado o vínculo de X
   com B (único desse estilo lá), B some do filtro.

### V9: Recuperação de senha (FR-003a) 🧪
1. Com Bruno logado em duas abas, peça `/esqueci-senha` numa janela anônima. **Esperado**:
   mensagem neutra; link no Mailpit. E-mail inexistente → mesma mensagem, nenhum e-mail.
2. Redefina pelo link. **Esperado**: as abas perdem a sessão; o link reusado → `400
   invalid_token`; login com a nova senha funciona.

### V10: Equipe do estúdio e registro (FR-021 a FR-024) 🧪
1. Ana convida bruno@… (já tem conta) e um e-mail sem conta. **Esperado**: e-mails com
   `/convite-equipe/...`; Bruno aceita logado e o Estúdio A aparece no seletor de área dele;
   o outro link leva ao cadastro com o e-mail pré-preenchido.
2. Bruno abre um link de convite enviado a outro e-mail. **Esperado**: `409 invitation_not_valid`.
3. Como membro, Bruno cria um avulso e altera os próprios dias de atendimento. **Esperado**:
   permitido; as telas `equipe`, `registro` e `perfil` não aparecem para ele; `PATCH
   /studios/{id}` com `seats` → `403`.
4. Ana abre `registro`. **Esperado**: as duas ações de Bruno com nome, data e o que mudou;
   filtro por membro e por período funciona.
5. Notificação in-app de um pedido novo aparece para Ana e Bruno na área do estúdio; o
   e-mail vai só para Ana.
6. Ana remove Bruno com o painel dele aberto. **Esperado**: a próxima ação dele no estúdio →
   `403`, a UI tira o estúdio do seletor; a área Tatuador e "Meus agendamentos" seguem.
7. Ana tenta criar um segundo estúdio. **Esperado**: `409 already_owns_studio`.

### V11: Privacidade e exclusões (FR-018 a FR-020a) 🧪
1. Confira `/studios/{slug}`, `/artists/{slug}`, `/search/*` e `/studios/{slug}/availability`.
   **Esperado**: nenhum e-mail, telefone, descrição ou imagem de cliente.
2. Davi exclui o **Estúdio B** (sem excluir a conta). **Esperado**: agendamentos futuros de B
   cancelados com notificação; vínculos encerrados; B some das buscas; Davi continua com o
   perfil de tatuador Z.
3. Carla (Y) exclui a conta com um agendamento futuro. **Esperado**: agendamento cancelado,
   cliente e estúdio notificados; `/tatuadores/tatuador-y` → 404; imagens removidas do MinIO;
   histórico do estúdio mantém o nome artístico.
4. Teste com relógio 12 meses após um agendamento concluído. **Esperado**: contato,
   descrição e imagens anonimizados; data, estúdio e tatuador mantidos.

### V12: Limite de pendentes e remarcação (FR-011c, FR-011d) 🧪
1. Dois pedidos com o mesmo e-mail em estúdios diferentes → ambos `pending`; um terceiro (ou
   outro e-mail com o mesmo telefone em outro formato) → `409 pending_limit_reached`.
2. 10 pedidos simultâneos com o mesmo e-mail → no máximo 2 `201`.
3. Pelo link do visitante, remarque um confirmado. **Esperado**: volta a `pending` no novo
   horário; o antigo reaparece; X e estúdio notificados. Para um horário lotado → `409
   slot_full` e nada muda.

### V13: Convites de vínculo e verificação (FR-004a, FR-001a) 🧪
1. Ana convida W e cancela. **Esperado**: some da lista de W; novo convite permitido.
2. Convite de vínculo com mais de 24h → `expired`, estúdio notificado; aceitar → `409
   invitation_not_valid`.
3. Conta nova com perfil de tatuador **sem** confirmar o e-mail, vinculada ao Estúdio A.
   **Esperado**: banner no painel; perfil → 404; não aparece em buscas, no estúdio nem para
   agendar. Confirmado o e-mail, aparece em todos.

### V14: Responsividade
Em 390×844 e 1440×900, percorra V1–V13. **Esperado**: sem rolagem horizontal, navegação
inferior no mobile e lateral no desktop, seletor de área utilizável nos dois, alvos ≥ 44 px.
