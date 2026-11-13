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
| `/agendamentos/acesso/:token` | Agendamento do visitante: status, detalhes, remarcar e cancelar | — | — |
| `/entrar`, `/cadastro`, `/verificar-email` | Autenticação; o cadastro pede só nome, e-mail, telefone e senha | — | — |
| `/esqueci-senha`, `/redefinir-senha/:token` | Recuperação de senha (FR-003a); a solicitação sempre mostra "se o e-mail existir, enviamos um link" | — | — |
| `/convite-equipe/:token` | Convite de membro (FR-021): logado com o e-mail certo → aceitar; sem conta → cadastro com e-mail pré-preenchido e aceite em seguida | — | — |

### Fluxo de agendamento (US2, FR-009)
1. Entrada pelo **estúdio** → escolhe o tatuador vinculado. Entrada pelo **tatuador**
   (`?tatuador=:slug`) → escolhe o estúdio entre os vínculos ativos. Sem os dois definidos,
   **não** há horários na tela.
2. Calendário só com os dias que têm vaga, dentro da janela de antecedência do estúdio e nos
   dias em que o tatuador atende ali → lista de horários livres.
3. Pedido (FR-008a): descrição (contador até 1000), local do corpo, tamanho (P/M/G com a
   faixa em cm) e até 3 imagens com prévia; no mobile, câmera direto (`accept="image/*"`).
4. Dados: logado → confirmação com os dados da conta; visitante → nome, e-mail e telefone.
   Erro `pending_limit_reached` explica que é preciso aguardar ou cancelar um pedido.
5. Confirmação: "Pedido enviado: aguardando aprovação do tatuador (até 48h)".

### Remarcação (FR-011d)
Em "Meus agendamentos" e no link do visitante, "Remarcar" abre o passo 2 com estúdio e
tatuador fixos; ao confirmar, o pedido volta para "aguardando aprovação". Em `409`, mostra a
mensagem, mantém o horário original e atualiza a lista.

Cancelar e remarcar só aparecem enquanto o horário não começou (FR-011).

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
| `/painel/tatuador/agenda` | Agenda consolidada de todos os estúdios, cada agendamento com o estúdio; cancelar até o início (US4) |
| `/painel/tatuador/pedidos` | Pendentes com prazo, detalhes do pedido e imagens ampliáveis; marca "remarcado"; aprovar ou recusar |
| `/painel/tatuador/estudios` | Convites (aceitar/recusar, com tempo restante de 24h), vínculos ativos com os dias de atendimento (somente leitura) e "encerrar vínculo" |
| `/painel/tatuador/portfolio` | Upload, edição, remoção e ordenação de trabalhos |
| `/painel/tatuador/perfil` | Nome artístico, bio, cidade, estilos, foto |

### Área Estúdio (`/painel/estudio/:studioId/...`)
Itens marcados com 👑 só aparecem para o dono (FR-022).

| Rota | Tela |
|---|---|
| `agenda` | Agendamentos do estúdio, filtráveis por tatuador; semana (desktop) / dia (mobile); cancelar |
| `horarios` | Padrão semanal, exceções por data, horários avulsos, janela de antecedência; macas visíveis para todos e editáveis 👑 |
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
