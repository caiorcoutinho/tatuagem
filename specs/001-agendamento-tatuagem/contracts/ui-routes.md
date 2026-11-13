# Contrato de UI: rotas e comportamento responsivo

SPA React servida pelo nginx. Todas as telas MUST funcionar em **mobile (≥ 360 px)** e
**desktop (≥ 1024 px)**; abordagem mobile-first com breakpoints Tailwind (`sm` 640, `md`
768, `lg` 1024). Validação e2e em 390×844 e 1440×900.

## Rotas públicas

| Rota | Tela | Mobile | Desktop |
|---|---|---|---|
| `/` | Busca de tatuadores e estúdios (nome, cidade, estilo) | filtros em painel inferior (bottom sheet) | filtros em barra lateral |
| `/tatuadores/:slug` | Perfil + portfólio em grade + estúdios onde atende (FR-016) | grade com 2 colunas | grade com 4–5 colunas |
| `/tatuadores/:slug/trabalhos/:id` | Trabalho ampliado + "Agendar" (US3) | tela cheia, deslizar entre trabalhos | modal sobre a grade |
| `/estudios/:slug` | Perfil do estúdio + tatuadores vinculados | lista empilhada | grade de cards |
| `/agendar/:studioSlug` | Fluxo de agendamento (ver abaixo) | assistente com 1 passo por tela | passos lado a lado |
| `/agendamentos/acesso/:token` | Agendamento do visitante: status, data/faixa pedida ou início e duração confirmados, detalhes, remarcar e cancelar | — | — |
| `/agendamentos/reenviar-link` | "Reenviar meu link": informa o e-mail e recebe os links dos agendamentos ativos; mensagem sempre neutra (FR-002b) | — | — |
| `/entrar`, `/cadastro`, `/verificar-email` | Autenticação; o cadastro pede só nome, e-mail, telefone e senha | — | — |
| `/esqueci-senha`, `/redefinir-senha/:token` | Recuperação de senha (FR-003a); a solicitação sempre mostra "se o e-mail existir, enviamos um link" | — | — |
| `/convite-equipe/:token` | Convite de membro (FR-021): logado com o e-mail certo → aceitar; sem conta → cadastro com e-mail pré-preenchido e aceite em seguida | — | — |

### Fluxo de agendamento (US2, FR-009)
1. Entrada pelo **estúdio** → escolhe o tatuador vinculado. Entrada pelo **tatuador**
   (`?tatuador=:slug`) → escolhe o estúdio entre os vínculos ativos. Sem os dois definidos,
   **não** há horários na tela.
2. Calendário só com as datas oferecidas (janela de antecedência, dias em que o tatuador
   atende ali e ao menos 30 min livres) → escolha de **uma faixa** da data (ex.: "14:00–18:00").
   Texto de apoio: "O tatuador define o horário exato e a duração ao aprovar".
3. Pedido (FR-008a): descrição (contador até 1000), local do corpo, tamanho (P/M/G com a
   faixa em cm) e até 3 imagens com prévia; no mobile, câmera direto (`accept="image/*"`).
4. Dados: logado → confirmação com os dados da conta; visitante → nome, e-mail e telefone.
   Erro `pending_limit_reached` explica que é preciso aguardar ou cancelar um pedido.
5. Confirmação: "Pedido enviado: aguardando aprovação do tatuador (até 48h)". Para
   visitante, mostra o **link de acesso** com botão "Copiar link" e o aviso "guarde este
   link: é por ele que você acompanha o pedido" (FR-002b).

### Remarcação (FR-011d)
Em "Meus agendamentos" e no link do visitante, "Remarcar" abre o passo 2 com estúdio e
tatuador fixos; ao confirmar, o pedido volta para "aguardando aprovação" com a nova data e
faixa. Em `409`, mostra a mensagem e mantém o agendamento como estava.

Cancelar aparece para pedidos pendentes e para sessões confirmadas que ainda não começaram;
remarcar, para pendentes antes da data pedida e confirmadas antes do início (FR-011, FR-011d).

Estados obrigatórios: carregando (skeleton), vazio ("nenhum horário disponível"), tatuador
sem vínculos ("não atende em nenhum estúdio no momento"), erros `409` com mensagem própria.

## Painel (`/painel`): áreas da conta (FR-003)

O topo do painel tem um **seletor de área**, montado a partir de `GET /auth/me` → `areas`:
"Meus agendamentos" (sempre), "Tatuador" (se tiver perfil), o estúdio próprio (se houver) e
cada estúdio em que é membro. A área escolhida é lembrada no navegador. Sem perfil de
tatuador nem estúdio, o painel oferece "Criar perfil de tatuador" e "Criar meu estúdio".

### Área pessoal
| Rota | Tela |
|---|---|
| `/painel/agendamentos` | Meus agendamentos como cliente, com detalhes; remarcar e cancelar |
| `/painel/conta` | Nome, telefone, e-mail (verificação), senha; "Excluir conta" com confirmação de senha e lista do que será afetado (estúdio próprio, perfil de tatuador, agendamentos futuros) (FR-020) |
| `/painel/notificacoes` | Notificações pessoais |

### Área Tatuador
| Rota | Tela |
|---|---|
| `/painel/tatuador/agenda` | Agenda consolidada de todos os estúdios, cada sessão com o estúdio; cancelar até o início; "Não compareceu" de 0 a 7 dias após o início, com desfazer (US4, FR-011e) |
| `/painel/tatuador/pedidos` | Pendentes com prazo, data e faixa pedidas, detalhes e imagens ampliáveis; marca "remarcado". **Aprovar** abre um seletor com os intervalos livres da data (blocos de 30 min) para escolher início e duração (FR-008b); recusar |
| `/painel/tatuador/estudios` | Convites (aceitar/recusar, com tempo restante de 24h), vínculos ativos com os dias de atendimento (somente leitura) e "encerrar vínculo" |
| `/painel/tatuador/portfolio` | Upload, edição, remoção e ordenação de trabalhos |
| `/painel/tatuador/perfil` | Nome artístico, bio, cidade, estilos, foto |

### Área Estúdio (`/painel/estudio/:studioId/...`)
Itens marcados com 👑 só aparecem para o dono (FR-022).

| Rota | Tela |
|---|---|
| `agenda` | Sessões confirmadas e pedidos pendentes do estúdio, filtráveis por tatuador; semana (desktop) / dia (mobile); cancelar; "Não compareceu" (FR-011e); selo "e-mail do cliente não entregue" com o telefone em destaque (FR-002b) |
| `horarios` | Faixas de atendimento: padrão semanal, exceções por data, faixas avulsas, janela de antecedência; macas visíveis para todos e editáveis 👑 |
| `tatuadores` | Convidar, convites pendentes com tempo restante e cancelar, vínculos ativos com 7 caixas de dias de atendimento, encerrar |
| `notificacoes` | Notificações do estúdio, compartilhadas pela equipe (FR-024) |
| `equipe` 👑 | Membros, convites pendentes, convidar por e-mail, remover |
| `registro` 👑 | Registro de ações com filtros por membro e período (FR-023) |
| `perfil` 👑 | Perfil público, foto, macas, fuso; "Excluir estúdio" com confirmação de senha (FR-020a) |

Membro removido que tenta agir recebe `403`: a UI mostra "Você não tem mais acesso a este
estúdio", remove o estúdio do seletor e volta para a área pessoal.

Contas com e-mail não verificado veem um banner fixo no painel: "Seu perfil e seu estúdio
não aparecem publicamente até você confirmar o e-mail", com "Reenviar link" (FR-001a).

Navegação: barra inferior com ícones no mobile; barra lateral no desktop; o seletor de área
fica no cabeçalho (menu suspenso no mobile). Contador de notificações não lidas da área
atual no cabeçalho.

## Acessibilidade mínima
Alvos de toque ≥ 44 px, contraste AA, foco visível, `alt` obrigatório (título do trabalho)
nas imagens do portfólio, formulários com `label`.
