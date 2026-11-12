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
docker compose up -d --build          # nginx, backend, worker, postgres, minio, mailpit
docker compose exec backend alembic upgrade head
docker compose exec backend python -m app.seed   # estilos de tatuagem + dados de demonstração
```

| Serviço | URL |
|---|---|
| Aplicação (nginx) | http://localhost:8080 |
| API / OpenAPI | http://localhost:8080/api/v1/docs |
| Mailpit (e-mails) | http://localhost:8025 |
| Console MinIO | http://localhost:9001 |

Esperado: `GET http://localhost:8080/api/v1/health` → `{"status":"ok","db":"ok","storage":"ok"}`.

## Testes automatizados

```bash
docker compose -f compose.test.yaml run --rm backend-tests   # pytest (unit + integração com Postgres real)
cd frontend && npm ci && npm test                            # Vitest
cd frontend && npx playwright test                           # e2e em mobile (390×844) e desktop (1440×900)
pre-commit run --all-files                                   # ruff, eslint, prettier
```

Esperado: tudo verde. Os testes de integração cobrem os cenários marcados com 🧪 abaixo.

## Cenários de validação manual

Use a UI em http://localhost:8080 no desktop e no modo responsivo do navegador (iPhone 12).

### V1: Estúdio, vínculo e horários (US1) 🧪
1. Cadastre o **Estúdio A** (fuso `America/Sao_Paulo`) e o **Tatuador X**.
2. Em `/painel/tatuadores`, convide X. **Esperado**: X recebe e-mail no Mailpit e
   notificação in-app; X ainda não aparece em `/estudios/estudio-a`.
3. X aceita em `/painel/estudios`. **Esperado**: X aparece no perfil do estúdio.
4. Em `/painel/horarios`, crie o padrão seg–sex 10:00–14:00, capacidade 2.
   **Esperado**: ao agendar com X no estúdio, os horários dos próximos dias úteis aparecem.
5. Bloqueie a próxima segunda. **Esperado**: a data some da disponibilidade.

### V2: Agendamento como visitante e aprovação (US2) 🧪
1. Em aba anônima, abra `/tatuadores/tatuador-x` → "Agendar". **Esperado**: é preciso
   escolher o estúdio antes de ver horários.
2. Escolha um horário e informe nome, e-mail e telefone. **Esperado**: status "pendente";
   e-mail com link `/agendamentos/acesso/...` no Mailpit; X é notificado.
3. X aprova em `/painel/pedidos`. **Esperado**: status `confirmed`; cliente e estúdio
   recebem e-mail.
4. Pelo link do e-mail, cancele. **Esperado**: status `cancelled`; vaga volta a aparecer;
   X e estúdio são notificados.

### V3: Capacidade e concorrência (FR-007a, FR-010, SC-003) 🧪
1. Vincule também o **Tatuador Y** e o **Tatuador Z** ao Estúdio A (capacidade 2).
2. Agende X e Y no mesmo horário. **Esperado**: o horário some para Z (`slot_full`).
3. Vincule X ao **Estúdio B** com horário sobreposto. **Esperado**: em B, X não aparece livre
   naquele intervalo; forçar via API retorna `409 artist_unavailable`.
4. Teste automatizado dispara 20 requisições simultâneas para a última vaga.
   **Esperado**: exatamente 1 `201`, demais `409`.

### V4: Expiração (FR-011b) 🧪
1. Crie um pedido e não responda. Para validar sem esperar 48h, rode o teste de integração
   do worker com relógio controlado, ou ajuste `expires_at` no banco de dev para o passado.
2. **Esperado**: em até 60 s o status vira `expired`, a vaga é liberada e o cliente recebe
   e-mail.

### V5: Desvínculo e regras de edição (edge cases) 🧪
1. Com X confirmado no Estúdio A, encerre o vínculo. **Esperado**: o agendamento é mantido;
   X some do estúdio; novo agendamento retorna `409 affiliation_inactive`.
2. Tente excluir um horário avulso com agendamento ativo. **Esperado**: `409 slot_has_bookings`.

### V6: Cliente com conta (FR-002a) 🧪
1. Crie conta Cliente com o mesmo e-mail usado em V2 e verifique o e-mail.
   **Esperado**: o agendamento de visitante aparece em `/painel/agendamentos`.

### V7: Portfólio (US3, SC-004)
1. Como X, envie 3 imagens (JPEG, PNG, WebP) com título e estilo. **Esperado**: aparecem na
   grade de `/tatuadores/tatuador-x`; abrir um trabalho mostra a imagem ampliada e
   "Agendar".
2. Envie um `.txt` renomeado para `.jpg`. **Esperado**: `400 invalid_image`.
3. Com o seed de 100 trabalhos (`python -m app.seed --portfolio 100`), meça no DevTools
   (rede "Fast 4G"): a grade fica visível em < 2 s.

### V8: Agenda consolidada e busca (US4, US5)
1. Com X agendado nos estúdios A e B, abra `/painel/agenda` como X. **Esperado**: os dois
   agendamentos, cada um com o nome do estúdio.
2. Em `/`, filtre por estilo "fineline" e cidade. **Esperado**: só tatuadores
   correspondentes.

### V9: Responsividade
Em 390×844 e 1440×900, percorra V1–V8. **Esperado**: sem rolagem horizontal, navegação
inferior no mobile e lateral no desktop, alvos de toque ≥ 44 px.
