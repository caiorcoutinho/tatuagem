# Quickstart: validação ponta a ponta

**Feature**: `001-agendamento-tatuagem` · Referências: [API](./contracts/api.md),
[UI](./contracts/ui-routes.md), [modelo de dados](./data-model.md)

Este guia mostra como subir o ambiente e validar que a feature funciona. Ele não descreve
implementação.

## Pré-requisitos

- Docker + Docker Compose v2
- Node 22 LTS e Python 3.12 (só para rodar lint/testes fora dos containers)
- `cp .env.example .env` (valores de dev já preenchidos; nenhum segredo real). Em dev os
  e-mails vão por SMTP para o Mailpit (`MAIL_BACKEND=smtp`).
- **Só para validar o envio real (opcional)**: conta @gmail.com do projeto, app OAuth no
  Google Cloud em status "In production" com escopos `gmail.send` e `gmail.modify`, e
  `docker compose exec backend python -m app.tools.gmail_authorize` para gerar o
  `GMAIL_REFRESH_TOKEN`; depois `MAIL_BACKEND=gmail` (research.md R23).

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

Esperado: `GET http://localhost:8080/api/v1/health` → `{"status":"ok","db":"ok","storage":"ok","broker":"ok","mail":"ok","mail_sent_today":0,"outbox_backlog":0}`.

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

### V1: Estúdio, vínculo, dias e faixas (US1) 🧪
1. Como Ana, na área do Estúdio A → `tatuadores`, convide X (Bruno) e "Ana Tattoo".
   **Esperado**: convites com 24h restantes; ninguém aparece ainda em `/estudios/estudio-a`.
2. Bruno aceita em `/painel/tatuador/estudios`; Ana aceita o próprio convite na área
   Tatuador. **Esperado**: os dois aparecem no perfil público do estúdio.
3. Em `horarios`, crie as faixas seg–sex 10:00–14:00 e 14:00–18:00, capacidade 2, e mantenha
   a janela padrão (24h a 60 dias). **Esperado**: ao pedir com X, aparecem os dias úteis a
   partir de amanhã + 24h, até 60 dias, cada um com as duas faixas.
4. Em `tatuadores`, deixe X só às segundas e quartas (FR-007d). **Esperado**: com X só
   aparecem segundas e quartas; com "Ana Tattoo", todos os dias úteis. Bruno vê os dias em
   `/painel/tatuador/estudios` sem poder editar.
5. Bloqueie a próxima segunda. **Esperado**: a data some.
6. Mude a antecedência mínima para 0h e a máxima para 7 dias. **Esperado**: a data de hoje
   aparece (se ainda houver faixa por começar); nada além de 7 dias.

### V2: Pedido do visitante, aprovação com duração e link (US2, FR-008b, FR-002b) 🧪
1. Em aba anônima, abra `/tatuadores/tatuador-x` → "Agendar". **Esperado**: é preciso
   escolher o estúdio antes de ver datas.
2. Escolha uma quarta e a faixa 14:00–18:00, preencha descrição, local, tamanho, 2 imagens e
   nome, e-mail e telefone. **Esperado**: status "pendente"; a tela de confirmação mostra o
   link de acesso com "Copiar link"; o Mailpit recebe o e-mail `booking_received` com o mesmo
   link; X é notificado. Nada fica reservado: a faixa continua oferecida a outros clientes.
3. X abre o pedido em `/painel/tatuador/pedidos` → "Aprovar". **Esperado**: o seletor mostra
   os intervalos livres daquela quarta. Escolha início 12:00 e duração 4h (atravessa as duas
   faixas). **Esperado**: `confirmed` 12:00–16:00; cliente e Ana (dona) recebem e-mail com
   início e duração.
4. Tente aprovar outro pedido com duração de 45 min ou começando 09:00 (antes das faixas).
   **Esperado**: `400 validation_error` / `409 slot_unavailable`; pedido continua pendente.
5. Pelo link do e-mail, cancele. **Esperado**: `cancelled`; o intervalo volta a ficar livre;
   X e estúdio notificados.

### V3: Macas, capacidade e concorrência na aprovação (FR-007a, FR-007c, FR-010, SC-003) 🧪
1. Vincule também Y (Carla). Com três pedidos pendentes para a mesma quarta (X, "Ana Tattoo"
   e Y), aprove X e "Ana Tattoo" às 14:00–16:00. **Esperado**: ao aprovar Y no mesmo
   intervalo → `409 slot_full`; o seletor de Y não oferece 14:00–16:00.
2. Vincule X também ao Estúdio B (de Davi). Com X confirmado 12:00–16:00 em A, tente aprovar
   um pedido de X em B às 15:00. **Esperado**: `409 artist_unavailable`.
3. Crie uma faixa avulsa 13:00–17:00 (cap. 2) sobreposta ao padrão no Estúdio A e aprove
   sessões com inícios diferentes até ocupar as 2 macas. **Esperado**: a próxima aprovação
   sobreposta → `409 slot_full`, mesmo vinda da faixa avulsa.
4. Ana tenta reduzir as macas para 1. **Esperado**: `409 seats_below_bookings`.
5. Teste automatizado: 20 aprovações simultâneas (tatuadores diferentes) disputando a última
   maca, com intervalos sobrepostos de inícios diferentes. **Esperado**: exatamente 1 sucesso.

### V4: Expiração, conclusão, falta e lembretes (FR-011b, FR-011e, FR-012a) 🧪
Com relógio controlado nos testes de integração das tasks periódicas (ou ajustando datas no
banco de dev e disparando a task pelo Beat):
1. Pedido sem resposta por 48h, ou até o início da faixa pedida. Com antecedência mínima
   0h, faça às 10h um pedido para hoje na faixa 14:00–18:00. **Esperado**: o pedido nasce
   `pending` (não expirado) e vira `expired` às 14:00 se não for aprovado; cliente notificado.
2. Sessão confirmada que começa em 24h. **Esperado**: um único lembrete ao cliente com links
   de ver/remarcar/cancelar; nenhum para uma sessão confirmada faltando 3h.
3. Relógio às 18:00 (Brasília) com X tendo 2 sessões confirmadas amanhã, em A e B.
   **Esperado**: um único e-mail de resumo para Bruno listando as duas, cada uma no fuso do
   estúdio; reiniciar o `celery-beat` ou o `celery-worker` não reenvia. Carla, sem sessões
   amanhã, não recebe nada.
4. Uma hora após o início de uma sessão, Bruno marca "Não compareceu" e depois desfaz; marca
   de novo. **Esperado**: `no_show`; tentar marcar antes do início ou 8 dias depois →
   `409 no_show_window_closed`.
5. Sessão confirmada sem marcação, 7 dias após o início. **Esperado**: `completed`.

### V4b: Mensageria e outbox (research.md R6) 🧪
1. Teste de integração: uma ação que gera e-mail (ex.: aprovar pedido) cria a
   `notification` e a `outbox_message`; forçar rollback não deixa nenhuma das duas.
2. Pare o RabbitMQ (`docker compose stop rabbitmq`) e aprove um pedido. **Esperado**: a
   aprovação funciona; a `outbox_message` fica com `published_at` nulo. Suba o RabbitMQ: em
   segundos o relay publica e o e-mail chega ao Mailpit, uma única vez.
3. Pare o Mailpit e aprove outro pedido. **Esperado**: retentativas com backoff nos logs;
   esgotadas, a mensagem aparece em `email.dlq` e a notificação fica com `email_failed_at`;
   a agenda do estúdio mostra "e-mail do cliente não entregue" com o telefone em destaque.
4. Reentrega manual da mesma mensagem pela UI do RabbitMQ. **Esperado**: não reenvia.

### V5: Desvínculo, edição e cancelamento (edge cases) 🧪
1. Com X confirmado no Estúdio A, encerre o vínculo. **Esperado**: sessão mantida; pedidos
   pendentes de X em A não podem mais ser aprovados (`409 affiliation_inactive`); novos
   pedidos → `409 affiliation_inactive`.
2. Exclua uma faixa avulsa com sessão confirmada dentro. **Esperado**: `409 slot_has_bookings`.
3. Remova a quarta dos dias de X com uma sessão numa quarta. **Esperado**: permitido; a
   sessão continua; novas quartas somem para X.
4. Cancelar uma sessão confirmada já iniciada. **Esperado**: `409 booking_started`; botão
   oculto. Cancelar um pedido pendente: sempre permitido.

### V6: Conta que agenda como cliente (FR-002a) 🧪
1. Crie a conta **Eva** com o mesmo e-mail usado pelo visitante em V2 e verifique o e-mail.
   **Esperado**: o agendamento de visitante aparece em `/painel/agendamentos`.
2. Bruno (tatuador X) pede uma sessão com Y pela própria conta. **Esperado**: aceito; aparece
   em "Meus agendamentos" dele. Bruno tenta pedir com X → `409 self_booking`.

### V7: Portfólio (US3, SC-004)
1. Como X, envie 3 imagens (JPEG, PNG, WebP) com título e estilo. **Esperado**: aparecem na
   grade de `/tatuadores/tatuador-x`; abrir um trabalho mostra a imagem ampliada e "Agendar".
2. Envie um `.txt` renomeado para `.jpg`. **Esperado**: `400 invalid_image`.
3. Com o seed de 100 trabalhos (`python -m app.seed --portfolio 100`), meça no DevTools
   (rede "Fast 4G"): a grade fica visível em < 2 s.

### V8: Agenda consolidada e busca (US4, US5)
1. Com X confirmado em A e B, abra `/painel/tatuador/agenda` como Bruno. **Esperado**: as
   duas sessões, cada uma com o nome do estúdio; cancelar disponível até o início.
2. Em `/`, filtre tatuadores por estilo e cidade. **Esperado**: só correspondentes.
3. Filtre estúdios pelo estilo de X. **Esperado**: A e B aparecem; encerrado o vínculo de X
   com B (único desse estilo lá), B some do filtro.

### V9: Recuperação de senha (FR-003a) 🧪
1. Com Bruno logado em duas abas, peça `/esqueci-senha` numa janela anônima. **Esperado**:
   mensagem neutra; link no Mailpit. E-mail inexistente → mesma mensagem, nenhum e-mail.
2. Redefina pelo link. **Esperado**: as abas perdem a sessão; o link reusado → `400
   invalid_token`; login com a nova senha funciona.

### V10: Equipe do estúdio e registro (US6, FR-021 a FR-024) 🧪
1. Ana convida bruno@… (já tem conta) e um e-mail sem conta. **Esperado**: e-mails com
   `/convite-equipe/...`; Bruno aceita logado e o Estúdio A aparece no seletor de área dele;
   o outro link leva ao cadastro com o e-mail pré-preenchido.
2. Bruno abre um link de convite enviado a outro e-mail. **Esperado**: `409 invitation_not_valid`.
3. Como membro, Bruno cria uma faixa avulsa e altera os próprios dias de atendimento.
   **Esperado**: permitido; `equipe`, `registro` e `perfil` não aparecem para ele; `PATCH
   /studios/{id}` com `seats` → `403`.
4. Ana abre `registro`. **Esperado**: as duas ações de Bruno com nome, data e o que mudou;
   filtro por membro e por período funciona.
5. Notificação in-app de um pedido novo aparece para Ana e Bruno na área do estúdio; o
   e-mail vai só para Ana.
6. Ana remove Bruno com o painel dele aberto. **Esperado**: a próxima ação dele no estúdio →
   `403`, a UI tira o estúdio do seletor; a área Tatuador e "Meus agendamentos" seguem.
7. Ana tenta criar um segundo estúdio. **Esperado**: `409 already_owns_studio`.

### V11: Privacidade e exclusões (US7, FR-018 a FR-020a) 🧪
1. Confira `/studios/{slug}`, `/artists/{slug}`, `/search/*` e `/studios/{slug}/availability`.
   **Esperado**: nenhum e-mail, telefone, descrição ou imagem de cliente.
2. Davi exclui o **Estúdio B** (sem excluir a conta). **Esperado**: sessões futuras e pedidos
   de B cancelados com notificação; vínculos encerrados; B some das buscas; Davi continua com
   o perfil de tatuador Z.
3. Carla (Y) exclui a conta com uma sessão futura. **Esperado**: sessão cancelada, cliente e
   estúdio notificados; `/tatuadores/tatuador-y` → 404; imagens removidas do MinIO;
   histórico do estúdio mantém o nome artístico.
4. Teste com relógio 12 meses após uma sessão concluída. **Esperado**: contato, descrição e
   imagens anonimizados; data, estúdio e tatuador mantidos.

### V12: Limite de pendentes e remarcação (FR-011c, FR-011d) 🧪
1. Dois pedidos com o mesmo e-mail em estúdios diferentes → ambos `pending`; um terceiro (ou
   outro e-mail com o mesmo telefone em outro formato) → `409 pending_limit_reached`.
2. 10 pedidos simultâneos com o mesmo e-mail → no máximo 2 `201`.
3. Pelo link do visitante, remarque uma sessão confirmada para outra data e faixa.
   **Esperado**: volta a `pending` com a nova preferência; o intervalo antigo fica livre; X e
   estúdio notificados para nova aprovação.
4. Com o mesmo e-mail já tendo 2 pendentes, remarque uma sessão **confirmada** desse e-mail.
   **Esperado**: `409 pending_limit_reached` e nada muda.

### V13: Convites de vínculo e verificação (FR-004a, FR-001a) 🧪
1. Ana convida W e cancela. **Esperado**: some da lista de W; novo convite permitido.
2. Convite de vínculo com mais de 24h → `expired`, estúdio notificado; aceitar → `409
   invitation_not_valid`.
3. Conta nova com perfil de tatuador **sem** confirmar o e-mail, vinculada ao Estúdio A.
   **Esperado**: banner no painel; perfil → 404; não aparece em buscas, no estúdio nem para
   pedidos. Confirmado o e-mail, aparece em todos.

### V14: Recuperação do link do visitante (FR-002b) 🧪
1. Visitante faz um pedido, fecha a tela de confirmação sem copiar o link e apaga o e-mail.
   Em `/agendamentos/reenviar-link`, informa o e-mail. **Esperado**: mensagem neutra; um
   e-mail `guest_links` com links para todos os agendamentos ativos; o link antigo deixa de
   funcionar e o novo funciona.
2. Mesmo formulário com um e-mail sem agendamentos. **Esperado**: mesma mensagem; nenhum
   e-mail no Mailpit.
3. Com `MAIL_BACKEND=gmail` (opcional), faça um pedido com e-mail inexistente
   (`naoexiste-<aleatório>@gmail.com`). **Esperado**: em até ~10 min a varredura de
   devoluções marca a notificação e a agenda do estúdio mostra o aviso com o telefone.

### V15: Responsividade
Em 390×844 e 1440×900, percorra V1–V14. **Esperado**: sem rolagem horizontal, navegação
inferior no mobile e lateral no desktop, seletor de área e seletor de intervalo da aprovação
utilizáveis nos dois, alvos ≥ 44 px.
