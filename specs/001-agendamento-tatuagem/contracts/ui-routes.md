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
| `/agendamentos/acesso/:token` | Agendamento do visitante: status e cancelar | — | — |
| `/entrar`, `/cadastro`, `/verificar-email` | Autenticação; o cadastro escolhe o perfil (Estúdio, Tatuador ou Cliente) | — | — |

### Fluxo de agendamento (US2, FR-009)
1. Entrada pelo **estúdio** → escolhe o tatuador vinculado. Entrada pelo **tatuador**
   (`?tatuador=:slug`) → escolhe o estúdio entre os vínculos ativos. Sem os dois definidos,
   **não** há horários na tela.
2. Calendário com dias que têm vaga → lista de horários livres.
3. Dados: logado como cliente → confirmação direta; visitante → nome, e-mail e telefone.
4. Confirmação: "Pedido enviado: aguardando aprovação do tatuador (até 48h)".

Estados obrigatórios: carregando (skeleton), vazio ("nenhum horário nos próximos 60 dias"),
tatuador sem vínculos ("não atende em nenhum estúdio no momento"), erros `409` com mensagem
própria e volta à lista de horários atualizada.

## Rotas autenticadas (`/painel`)

| Rota | Perfil | Tela |
|---|---|---|
| `/painel/agenda` | studio | Agendamentos do estúdio, filtráveis por tatuador (FR-013); semana (desktop) / dia (mobile) |
| `/painel/horarios` | studio | Padrão semanal, exceções por data e horários avulsos (FR-007b) |
| `/painel/tatuadores` | studio | Vínculos: convidar, pendentes, encerrar |
| `/painel/agenda` | artist | Agenda consolidada de todos os estúdios, cada agendamento identificado pelo estúdio (US4) |
| `/painel/pedidos` | artist | Pendentes com prazo de expiração; aprovar ou recusar |
| `/painel/estudios` | artist | Convites (aceitar/recusar) e vínculos ativos |
| `/painel/portfolio` | artist | Upload, edição, remoção e ordenação de trabalhos |
| `/painel/agendamentos` | client | Meus agendamentos e cancelamento |
| `/painel/perfil` | todos | Editar o perfil |
| `/painel/notificacoes` | todos | Notificações in-app |

Navegação: barra inferior com ícones no mobile; barra lateral no desktop. Contador de
notificações não lidas no cabeçalho.

## Acessibilidade mínima
Alvos de toque ≥ 44 px, contraste AA, foco visível, `alt` obrigatório (título do trabalho)
nas imagens do portfólio, formulários com `label`.
