# Feature Specification: Agendamento para Tatuadores e Estúdios

**Feature Branch**: `001-agendamento-tatuagem`

**Created**: 2026-11-12

**Status**: Draft

**Input**: User description: "crie uma aplicação para agendamentos de tatuadores e estúdios de tatuagem. são dois perfis principais: tatuador e estúdio, e o estúdio pode configurar agenda, disponibilizando horários, e vincular tatuadores. já o tatuador pode estar vinculado a diversos estúdios e colocar seu portfolio à mostra estilo pinterest/behance. um estúdio pode ter vários tatuadores e um tatuador pode tatuar em vários estúdios também. porém, as agendas dependem inteiramente dos estúdios, ou seja, não é possível agendar com o tatuador sem selecionar um estúdio onde ele atende, e não é possível agendar em um estúdio sem selecionar um tatuador."

## Clarifications

### Session 2026-11-12

- Q: Os horários são definidos por tatuador ou são gerais do estúdio? → A: Horários gerais
  do estúdio com capacidade fixa (ex.: número de macas), independente do tatuador; cada
  agendamento ocupa uma vaga e exige um tatuador vinculado livre.
- Q: O agendamento é confirmado automaticamente ou fica pendente? → A: Fica pendente até o
  tatuador aprovar ou recusar; a vaga fica reservada enquanto pendente e a recusa a libera.
- Q: O cliente precisa criar conta ou pode agendar como visitante? → A: Ambos: pode agendar
  como visitante (nome, e-mail, telefone) ou com conta opcional de perfil Cliente.
- Q: O que acontece com um agendamento pendente sem resposta? → A: Expira automaticamente
  em 48h ou no início do horário, o que vier antes; a vaga é liberada e o cliente notificado.
- Q: Como o estúdio cadastra os horários? → A: Padrão semanal recorrente + exceções
  (bloquear/editar datas específicas e criar horários avulsos extras).

### Session 2026-11-13

- Q: Como tratar horário avulso sobreposto a horários do padrão ou de exceções? → A: O
  estúdio tem um número fixo de macas; a soma de agendamentos ativos simultâneos no estúdio
  nunca ultrapassa esse número, mesmo com horários sobrepostos.
- Q: A recuperação de senha entra na v1? → A: Sim: link de uso único por e-mail, válido por
  1h; ao redefinir a senha, todas as sessões abertas são encerradas.
- Q: O que significa buscar estúdios por estilo? → A: Derivado: o estúdio corresponde ao
  estilo X se tiver ao menos um tatuador com vínculo ativo que trabalha nesse estilo.
- Q: Como tratar privacidade dos dados do cliente (LGPD)? → A: Contato visível só ao estúdio
  e ao tatuador do agendamento; anonimizado 12 meses após o fim do agendamento; qualquer
  usuário pode excluir a própria conta (perfil removido, agendamentos futuros cancelados com
  notificação).
- Q: Até quando um agendamento pode ser cancelado? → A: Por qualquer parte, até o início do
  horário; após o início, não é mais possível cancelar.
- Q: Como evitar que pedidos falsos travem a agenda? → A: Máximo de 2 agendamentos pendentes
  por e-mail e por telefone, somando todos os estúdios; o terceiro é recusado até um deles
  deixar de estar pendente.
- Q: O que o cliente informa sobre a tatuagem no pedido? → A: Descrição obrigatória (até 1000
  caracteres), local do corpo (lista), tamanho aproximado (P até 5 cm / M 5–15 cm / G acima
  de 15 cm) e até 3 imagens de referência, visíveis só ao estúdio e ao tatuador.
- Q: Estúdio e tatuador precisam verificar o e-mail? → A: Podem configurar tudo antes, mas só
  aparecem em páginas públicas, buscas e agendamentos depois de verificar o e-mail.
- Q: O que acontece com um convite de vínculo sem resposta? → A: Expira em 24h sem resposta
  (estúdio notificado); o estúdio também pode cancelar um convite pendente; depois disso pode
  convidar de novo.
- Q: Como remarcar um agendamento? → A: O cliente remarca para outro horário livre com o
  mesmo estúdio e tatuador; o agendamento volta para pendente e o tatuador aprova de novo.
- Q: Em que dias o tatuador atende em cada estúdio? → A: O estúdio define, para cada tatuador
  vinculado, os dias da semana em que ele atende ali; só esses dias aparecem para o cliente.
- Q: Com quanta antecedência dá para agendar? → A: Configurável por estúdio: mínimo de 0 a
  72h e máximo de 7 a 180 dias (padrão: 24h e 60 dias).
- Q: Haverá lembrete antes da sessão? → A: E-mail ao cliente 24h antes de agendamentos
  confirmados (com link para ver, remarcar ou cancelar) e resumo diário ao tatuador com os
  agendamentos do dia seguinte.
- Q: Mais de uma pessoa pode gerenciar o estúdio? → A: Sim: o dono convida membros por
  e-mail, que gerenciam agenda, vínculos e agendamentos; só o dono edita o perfil, as macas,
  os membros e exclui o estúdio; cada ação de gestão fica registrada (quem, quando, o quê) e
  visível ao dono. A conta passa a representar a pessoa (substitui o "perfil único por
  conta" das sessões anteriores): a mesma conta pode ter perfil de tatuador, ser dona de no
  máximo 1 estúdio, membro de vários e agendar como cliente.
- Q: Qual a política de backup dos dados? → A: Sem backup automatizado na v1.
- Q: Como tratar sessões mais longas que um horário? → A: A duração é definida pelo tatuador
  na aprovação: o cliente envia data e faixa de atendimento preferidas, nada fica reservado
  até o tatuador aprovar escolhendo início e duração; os "horários disponíveis" do estúdio
  passam a ser faixas de atendimento (períodos de funcionamento).
- Q: Como o visitante recupera o acesso ao pedido se perder o link? → A: O link aparece também
  na tela de confirmação (com botão de copiar); há uma página "reenviar meu link" que envia por
  e-mail os links dos pedidos ativos daquele e-mail, com resposta neutra; e, quando um e-mail
  ao cliente não puder ser entregue, estúdio e tatuador veem um aviso no agendamento com o
  telefone do cliente em destaque para contato.
- Q: Equipe e privacidade ganham histórias próprias, com qual prioridade? → A: Sim: US6
  Equipe do estúdio (P2) e US7 Privacidade e exclusão de conta (P1).
- Q: Qual provedor de e-mail e como detectar e-mail não entregue? → A: API do Gmail. No MVP,
  a partir de uma conta pessoal @gmail.com (não Workspace); devoluções detectadas lendo as
  mensagens de retorno na caixa da conta de envio.
- Q: Como registrar cliente que não comparece? → A: Tatuador ou estúdio podem marcar "não
  compareceu" de 0 a 7 dias após o início da sessão; sem marcação, ela vira "concluída"
  automaticamente ao fim desse prazo.
- Q: Quando expira um pedido pendente sem resposta? → A: Em 48h ou no início da faixa pedida
  (data + faixa escolhidas pelo cliente), o que vier antes; substitui a regra provisória
  "à 00h da data pedida".
- Q: Reduzir a quantidade de e-mails por agendamento por causa do limite do Gmail? → A: Não:
  mantém os e-mails por evento para todas as partes (~6 por sessão confirmada); o MVP é
  dimensionado para até ~3 mil agendamentos por mês.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Estúdio configura agenda e vincula tatuadores (Priority: P1)

O responsável pelo estúdio cria o perfil do estúdio, define as faixas de atendimento e vincula os tatuadores que atendem ali.

**Why this priority**: sem estúdio, vínculo e faixas de atendimento não existe agenda; é a base de todo o
fluxo de agendamento.

**Independent Test**: criar um estúdio, convidar um tatuador, ter o convite aceito e publicar
faixas de atendimento; verificar que as datas dessas faixas ficam disponíveis para pedidos com
o tatuador vinculado.

**Acceptance Scenarios**:

1. **Given** um estúdio cadastrado, **When** o estúdio convida um tatuador cadastrado,
   **Then** o tatuador recebe o convite e o vínculo só fica ativo após o aceite.
2. **Given** um estúdio com tatuador vinculado, **When** o estúdio cadastra faixas de
   atendimento informando a capacidade (sessões simultâneas), **Then** as datas e faixas
   ficam disponíveis para pedidos com os tatuadores vinculados que atendem naqueles dias,
   dentro da janela de antecedência do estúdio.
3. **Given** um tatuador vinculado, **When** o estúdio desfaz o vínculo, **Then** o tatuador
   deixa de aparecer como opção naquele estúdio e novos agendamentos com ele ali ficam
   bloqueados.
4. **Given** um estúdio com padrão semanal configurado, **When** ele bloqueia uma data
   (ex.: feriado) ou cria uma faixa avulsa, **Then** apenas aquela data é afetada e o padrão
   continua valendo para as demais semanas.
5. **Given** um tatuador vinculado, **When** o estúdio define que ele atende ali só às
   segundas e quartas, **Then** ao pedir uma sessão com esse tatuador no estúdio só aparecem
   datas de segunda e quarta.

---

### User Story 2 - Cliente agenda sessão escolhendo estúdio e tatuador (Priority: P1)

Um cliente escolhe um estúdio e um tatuador vinculado a ele (ou um tatuador e um estúdio
onde ele atende), escolhe a data e a faixa de atendimento preferidas e solicita a sessão; o
tatuador aprova definindo início e duração.

**Why this priority**: é o valor central do produto — transformar disponibilidade em sessão
marcada.

**Independent Test**: com um estúdio, um tatuador vinculado e faixas publicadas, um cliente
envia um pedido e o tatuador o aprova definindo início e duração; tentar pedir sem estúdio ou
sem tatuador deve ser impossível.

**Acceptance Scenarios**:

1. **Given** um estúdio com tatuadores vinculados e faixas de atendimento, **When** o
   cliente seleciona estúdio, tatuador, data e faixa preferidas e descreve a tatuagem
   (descrição, local do corpo, tamanho e, opcionalmente, imagens de referência), **Then** o
   pedido é registrado como pendente, **sem reservar vaga**, e o tatuador é notificado para
   aprovar ou recusar vendo os detalhes.
2. **Given** o cliente partindo do perfil de um tatuador, **When** ele inicia um agendamento,
   **Then** o sistema exige a escolha de um dos estúdios onde o tatuador está vinculado antes
   de exibir datas.
3. **Given** o cliente partindo do perfil de um estúdio, **When** ele inicia um agendamento,
   **Then** o sistema exige a escolha de um tatuador vinculado antes de exibir datas.
4. **Given** um tatuador com sessão confirmada num intervalo, **When** ele tenta aprovar
   outro pedido com intervalo sobreposto (em qualquer estúdio), **Then** o sistema impede a
   aprovação e o pedido continua pendente.
5. **Given** um agendamento existente, **When** cliente, tatuador ou estúdio o cancela,
   **Then** o intervalo (se estava confirmado) volta a ficar livre e as demais partes são
   notificadas.
6. **Given** um pedido pendente, **When** o tatuador aprova definindo início e duração
   dentro das faixas de atendimento da data pedida, **Then** o agendamento passa a confirmado
   com esse intervalo e o cliente e o estúdio são notificados; **When** o tatuador recusa,
   **Then** o pedido passa a recusado e o cliente é notificado.
7. **Given** um agendamento pendente ou confirmado, **When** o cliente o remarca para outra
   data e faixa com o mesmo estúdio e tatuador, **Then** o pedido volta a pendente com a nova
   preferência, o intervalo anterior (se confirmado) é liberado e o tatuador e o estúdio são
   notificados para uma nova aprovação.

---

### User Story 3 - Tatuador publica portfólio (Priority: P2)

O tatuador cria seu perfil e publica trabalhos em um portfólio visual em grade (estilo
mural de imagens), com títulos, descrições e estilos de tatuagem.

**Why this priority**: o portfólio é o principal fator de decisão do cliente, mas o
agendamento funciona sem ele.

**Independent Test**: um tatuador publica trabalhos e um visitante visualiza a grade e o
detalhe de cada trabalho, incluindo os estúdios onde o tatuador atende.

**Acceptance Scenarios**:

1. **Given** um tatuador cadastrado, **When** ele publica uma imagem com título e estilo,
   **Then** o trabalho aparece no portfólio público em grade.
2. **Given** um portfólio publicado, **When** um visitante abre um trabalho, **Then** vê a
   imagem ampliada, os detalhes e o atalho para agendar com o tatuador.
3. **Given** um trabalho publicado, **When** o tatuador o remove ou edita, **Then** o
   portfólio público reflete a mudança.

---

### User Story 4 - Tatuador gerencia vínculos e agenda consolidada (Priority: P2)

O tatuador aceita ou recusa convites de estúdios, pode se desvincular e visualiza em um único
lugar todos os seus agendamentos, de todos os estúdios.

**Why this priority**: permite ao tatuador atuar em vários estúdios sem conflito de horários.

**Independent Test**: tatuador vinculado a dois estúdios visualiza agendamentos de ambos numa
mesma agenda, identificados por estúdio.

**Acceptance Scenarios**:

1. **Given** um convite pendente, **When** o tatuador aceita, **Then** passa a aparecer como
   tatuador do estúdio.
2. **Given** um tatuador com agendamentos em dois estúdios, **When** abre sua agenda,
   **Then** vê todos os agendamentos com o estúdio de cada um.

---

### User Story 5 - Descoberta de estúdios e tatuadores (Priority: P3)

Visitantes buscam tatuadores e estúdios por nome, cidade e estilo de tatuagem.

**Why this priority**: melhora a aquisição, mas não é necessário para o fluxo básico.

**Independent Test**: buscar por um estilo e obter tatuadores com trabalhos daquele estilo.

**Acceptance Scenarios**:

1. **Given** tatuadores com estilos cadastrados, **When** o visitante filtra por estilo e
   cidade, **Then** apenas resultados correspondentes são exibidos.
2. **Given** um estúdio cujo único tatuador do estilo X encerra o vínculo, **When** o
   visitante filtra estúdios pelo estilo X, **Then** esse estúdio deixa de aparecer.

---

### User Story 6 - Equipe do estúdio (Priority: P2)

O dono do estúdio convida pessoas da equipe para ajudar na gestão (agenda, vínculos e
agendamentos) e acompanha tudo o que cada uma fez.

**Why this priority**: facilita a operação de estúdios com recepção ou sócios, mas o dono
consegue operar sozinho no início.

**Independent Test**: o dono convida um membro, o membro cria uma faixa avulsa e cancela um
agendamento, o dono vê as duas ações no registro e remove o membro, que perde o acesso na hora.

**Acceptance Scenarios**:

1. **Given** um estúdio, **When** o dono convida um e-mail e a pessoa aceita com a conta
   desse e-mail, **Then** ela passa a ver o estúdio no painel e pode gerenciar agenda,
   vínculos e agendamentos.
2. **Given** um membro, **When** ele tenta editar o perfil público, as macas ou a equipe,
   **Then** o sistema recusa.
3. **Given** ações feitas por membros, **When** o dono abre o registro de ações, **Then** vê
   autor, data/hora e o que mudou em cada uma, com filtro por membro e período.
4. **Given** um membro com o painel aberto, **When** o dono o remove, **Then** a próxima ação
   dele no estúdio é recusada, e a conta dele segue funcionando no resto.

---

### User Story 7 - Privacidade e exclusão de conta (Priority: P1)

Clientes têm seus dados de contato e detalhes do pedido protegidos e apagados após um ano;
qualquer pessoa pode excluir a própria conta, e o dono pode excluir o estúdio.

**Why this priority**: obrigatório antes de operar com dados reais de clientes (LGPD).

**Independent Test**: páginas públicas não mostram dados de clientes; ao excluir uma conta
com sessão futura, a sessão é cancelada e as partes são notificadas; um agendamento com mais
de 12 meses aparece anonimizado.

**Acceptance Scenarios**:

1. **Given** agendamentos existentes, **When** um visitante navega por páginas públicas e
   buscas, **Then** não vê nome, e-mail, telefone, descrição nem imagens de clientes.
2. **Given** um agendamento concluído há mais de 12 meses, **When** estúdio ou tatuador o
   consultam, **Then** veem data, estúdio, tatuador e status, mas contato, descrição e
   imagens foram apagados.
3. **Given** uma conta com sessão confirmada futura, **When** a pessoa exclui a conta
   confirmando a senha, **Then** a sessão é cancelada com notificação às demais partes e a
   conta deixa de existir publicamente.
4. **Given** um estúdio com sessões futuras, **When** o dono exclui o estúdio, **Then** as
   sessões são canceladas com notificação, os vínculos e a equipe são encerrados e o dono
   mantém a conta.

---

### Edge Cases

- Tatuador vinculado a dois estúdios não pode ter duas sessões confirmadas sobrepostas,
  mesmo em estúdios diferentes.
- Desvínculo de um tatuador com agendamentos futuros naquele estúdio: os agendamentos futuros
  são mantidos até serem cancelados explicitamente pelo estúdio ou tatuador; novos pedidos
  são bloqueados e pedidos pendentes nesse estúdio não podem mais ser aprovados.
- Estúdio exclui uma faixa avulsa que contém sessão confirmada: o sistema impede a
  exclusão até a sessão ser cancelada (para padrão semanal e datas bloqueadas, ver o
  caso de alteração do padrão abaixo).
- Duas aprovações simultâneas que usariam a última maca ou capacidade de uma faixa, ou o
  mesmo tatuador em intervalos sobrepostos: apenas uma é concluída; a outra recebe aviso e o
  pedido continua pendente.
- Tatuador sem nenhum vínculo ativo: perfil e portfólio continuam visíveis, mas o botão de
  agendar indica que não há estúdios disponíveis.
- Pedido para data passada ou aprovação com início no passado não são permitidos.
- Dono remove um membro que está com o painel do estúdio aberto: a próxima ação dele naquele
  estúdio é recusada e o estúdio some do painel dele; o restante da conta segue funcionando;
  as ações que ele já fez continuam no registro com o nome dele.
- Tatuador que também é membro do estúdio altera os próprios dias de atendimento ali: é
  permitido (age como membro) e fica no registro de ações.
- Tatuador tenta agendar consigo mesmo: o sistema recusa.
- Conta que já é dona de um estúdio tenta criar outro: o sistema recusa (máximo 1).
- Agendamento cancelado ou remarcado depois do lembrete de 24h: nenhum lembrete adicional
  para o intervalo antigo; se o novo intervalo for confirmado com mais de 24h de antecedência,
  recebe o próprio lembrete.
- Cliente tenta pedir ou remarcar para uma data fora da janela de antecedência do estúdio
  (ex.: hoje com mínimo de 24h, ou daqui a 90 dias com máximo de 60): a data não é oferecida
  e, se enviada diretamente, é recusada.
- Estúdio ou tatuador com e-mail não verificado: o painel mostra um aviso com opção de
  reenviar o link; o perfil não aparece publicamente e não recebe agendamentos. Um estúdio
  verificado pode convidar um tatuador não verificado, mas o tatuador só aparece como opção
  de agendamento depois de verificar.
- Tatuador tenta aprovar com início ou duração fora das faixas de atendimento da data, ou
  num intervalo sem maca/capacidade livre ou em que ele já tem sessão: a aprovação é recusada
  com a indicação dos intervalos livres daquela data; o pedido continua pendente.
- Estúdio remove um dia de atendimento de um tatuador que já tem agendamentos ativos nesse
  dia: a alteração é permitida, os agendamentos existentes são mantidos e apenas novos
  agendamentos naquele dia ficam bloqueados.
- Visitante digita o e-mail errado: vê o link na tela de confirmação e pode salvá-lo; se fechar
  a tela, o pedido segue, e o estúdio e o tatuador veem o aviso de e-mail não entregue com o
  telefone para contato (se o envio falhar).
- Alguém usa "reenviar meu link" com um e-mail que não tem agendamentos: recebe a mesma
  mensagem neutra e nenhum e-mail é enviado.
- Limite diário de envio de e-mails atingido: os e-mails ficam na fila e saem quando a cota
  é liberada, na ordem em que foram gerados; as notificações dentro da aplicação continuam
  imediatas.
- Tentativa de marcar "não compareceu" antes do início da sessão ou mais de 7 dias depois:
  o sistema recusa; após 7 dias a sessão já está concluída e o status não muda mais.
- Tatuador tenta aceitar um convite expirado ou cancelado: o sistema recusa e informa que o
  convite não é mais válido.
- Cliente envia imagem de referência inválida (formato não aceito, arquivo corrompido ou
  acima de 10 MB) ou mais de 3 imagens: o pedido é recusado com indicação do problema e
  não é registrado.
- Cliente com 2 pedidos pendentes tenta um terceiro (mesmo e-mail ou mesmo telefone): o
  sistema recusa e informa que é preciso aguardar a resposta ou cancelar um dos pedidos.
- Tentativa de cancelar uma sessão confirmada que já começou: o sistema recusa e informa
  que a sessão já iniciou.
- Tatuador não responde a um pedido pendente: o pedido expira em 48h ou no início da faixa
  pedida (o que vier antes) e o cliente é notificado. Ex.: pedido feito hoje às 10h para hoje
  na faixa 14:00–18:00 expira às 14:00 se não for aprovado antes.
- Estúdio sem tatuadores vinculados não aceita agendamentos.
- Data em que o estúdio não tem mais nenhum intervalo livre de 30 min (macas ou capacidade
  tomadas por sessões confirmadas) deixa de ser oferecida para pedidos.
- Estúdio altera o padrão semanal ou bloqueia uma data com sessões confirmadas: as sessões
  são mantidas até serem canceladas; pedidos pendentes para uma data que ficou sem faixas não
  podem ser aprovados (o tatuador recusa ou o pedido expira).
- Estúdio reduz a capacidade de uma faixa abaixo do pico de sessões confirmadas nela: o
  sistema impede a alteração até que sessões sejam canceladas.
- Faixas sobrepostas (ex.: avulsa sobre faixa do padrão): sessões podem ser aprovadas em
  qualquer delas, mas a soma de sessões confirmadas simultâneas no estúdio não passa do
  número de macas.
- Usuário exclui a conta com agendamentos futuros: os agendamentos são cancelados e as
  demais partes notificadas antes da remoção; agendamentos passados permanecem no histórico
  das outras partes, sem dados pessoais de quem excluiu (exceto o nome do estúdio ou
  tatuador, que é público).
- Estúdio reduz o número de macas abaixo do pico de sessões confirmadas simultâneas futuras:
  o sistema impede a alteração até que agendamentos sejam cancelados.
- Na aprovação, o tatuador só pode escolher intervalos em que não tenha outra sessão
  confirmada (em qualquer estúdio).

## Requirements *(mandatory)*

### Functional Requirements

**Contas e perfis**

- **FR-001**: O sistema MUST permitir o cadastro de contas (nome, e-mail, senha, telefone).
  A partir da conta, a pessoa MAY criar um perfil de Tatuador e MAY criar um Estúdio, do qual
  passa a ser a dona. Estúdio e Tatuador têm página pública (nome, descrição, foto,
  cidade/endereço e número de macas para estúdio, estilos para tatuador).
- **FR-001a**: Toda conta MUST receber um link de verificação de e-mail no cadastro, com
  opção de reenvio. Tatuadores e estúdios cujo dono tem e-mail não verificado MAY configurar
  perfil, faixas de atendimento, vínculos e portfólio, mas MUST NOT aparecer em páginas públicas, buscas
  ou na disponibilidade para agendamento até o e-mail ser verificado.
- **FR-002**: O sistema MUST permitir que clientes solicitem agendamentos como visitante,
  informando nome, e-mail e telefone; o visitante MUST poder acompanhar e cancelar o
  agendamento por um link de acesso enviado ao e-mail informado.
- **FR-002a**: Qualquer conta MAY agendar como cliente (inclusive quem tem perfil de
  tatuador ou gerencia estúdio), exceto um tatuador consigo mesmo; com conta, o cliente MUST
  visualizar todos os seus agendamentos em uma área própria. Agendamentos feitos como visitante com o
  mesmo e-mail MUST ser associados à conta quando o e-mail da conta for verificado.
- **FR-002b**: Após enviar o pedido, a tela de confirmação MUST exibir o link de acesso do
  visitante com opção de copiar. O sistema MUST oferecer uma página "reenviar meu link" em que
  o visitante informa o e-mail e recebe nele os links de todos os seus agendamentos ativos; a
  resposta MUST ser a mesma exista ou não algum agendamento para aquele e-mail. Quando um
  e-mail destinado ao cliente não puder ser entregue (tentativas esgotadas ou mensagem de
  devolução recebida), o agendamento
  MUST exibir ao estúdio e ao tatuador um aviso "e-mail do cliente não entregue" com o
  telefone do cliente em destaque.
- **FR-003**: Cada conta representa uma pessoa e MAY, ao mesmo tempo: ter um perfil de
  Tatuador (no máximo um), ser dona de no máximo 1 estúdio, ser membro de vários estúdios e
  agendar como cliente. Cada estúdio tem exatamente um dono. O painel MUST permitir alternar
  entre as áreas da conta (tatuador, cada estúdio gerenciado, meus agendamentos).
- **FR-003a**: O usuário MUST poder redefinir a senha por um link de uso único enviado ao
  e-mail da conta, válido por 1h. Ao redefinir, todas as sessões abertas da conta MUST ser
  encerradas. A solicitação MUST responder da mesma forma para e-mails cadastrados ou não.

**Vínculos**

- **FR-004**: O estúdio MUST poder convidar tatuadores cadastrados; o vínculo MUST ficar ativo
  apenas após aceite do tatuador.
- **FR-004a**: Um convite sem resposta MUST expirar 24h após o envio, com notificação ao
  estúdio; o estúdio MUST poder cancelar um convite pendente. Após expiração, cancelamento ou
  recusa, o estúdio MAY convidar o mesmo tatuador novamente.
- **FR-005**: Estúdio e tatuador MUST poder encerrar um vínculo a qualquer momento.
- **FR-006**: Um tatuador MUST poder ter vínculos ativos com vários estúdios, e um estúdio com
  vários tatuadores.

**Agenda**

- **FR-007**: Apenas o estúdio MUST poder criar, editar e remover faixas de atendimento
  (formerly referred to as "horários disponíveis"): períodos em que o estúdio atende. Cada
  faixa é do estúdio (não de um tatuador) e MUST ter capacidade ≥ 1 (máximo de sessões
  simultâneas nela).
- **FR-007a**: Em nenhum instante o número de sessões confirmadas dentro de uma faixa MAY
  ultrapassar a capacidade dessa faixa.
- **FR-007b**: O estúdio MUST poder definir um padrão semanal recorrente (dias da semana,
  faixas de horário e capacidade) que gera as faixas de atendimento, e MUST poder registrar
  exceções: bloquear ou editar as faixas de datas específicas e criar faixas avulsas.
- **FR-007c**: O estúdio MUST informar seu número de macas (≥ 1). Em nenhum instante o
  número de sessões confirmadas simultâneas do estúdio MAY ultrapassar o número de macas,
  mesmo quando faixas (padrão, exceção ou avulsa) se sobrepõem. A capacidade de uma faixa
  MUST ser ≤ número de macas.
- **FR-007d**: O estúdio MUST definir, para cada tatuador com vínculo ativo, os dias da
  semana em que ele atende no estúdio (padrão ao aceitar o vínculo: todos os dias). Só datas
  nesses dias (no fuso do estúdio) MUST ser oferecidas para pedidos e aceitas na aprovação
  com aquele tatuador naquele estúdio. O tatuador MUST ver esses dias, mas só pode editá-los
  se também for dono ou membro do estúdio.
- **FR-007e**: O estúdio MUST poder configurar a antecedência mínima (0 a 72h, padrão 24h) e
  a máxima (7 a 180 dias, padrão 60 dias) para pedidos. Só datas dentro dessa janela,
  contada a partir do momento do pedido, MUST ser oferecidas e aceitas, tanto em novos
  pedidos quanto em remarcações; na aprovação, o início MUST ser no futuro. Alterar a janela não afeta agendamentos já
  existentes.
- **FR-008**: Todo agendamento MUST referenciar exatamente um estúdio, um tatuador com vínculo
  ativo nesse estúdio e uma data e uma faixa de atendimento preferidas desse estúdio; quando
  confirmado, MUST ter também o início e a duração definidos pelo tatuador.
- **FR-008b**: Ao aprovar, o tatuador MUST definir início e duração (múltiplos de 30 min,
  mínimo 30 min) na data pedida. O intervalo MUST estar inteiramente dentro das faixas de
  atendimento do estúdio naquela data (podendo atravessar faixas contíguas), pode começar
  fora da faixa preferida pelo cliente e MUST respeitar FR-007a, FR-007c, FR-007d e FR-010.
  O sistema MUST mostrar ao tatuador os intervalos livres da data para facilitar a escolha.
- **FR-008a**: Todo pedido de agendamento MUST conter: descrição da tatuagem (obrigatória,
  até 1000 caracteres), local do corpo (obrigatório, de uma lista fixa), tamanho aproximado
  (obrigatório: P até 5 cm, M 5–15 cm, G acima de 15 cm) e MAY conter até 3 imagens de
  referência (JPEG, PNG ou WebP, até 10 MB cada). Os detalhes e as imagens MUST ser visíveis
  apenas ao estúdio e ao tatuador do agendamento e ao próprio cliente.
- **FR-009**: O sistema MUST impedir agendamentos sem estúdio ou sem tatuador selecionado.
- **FR-010**: O sistema MUST impedir sobreposição de sessões confirmadas para o mesmo
  tatuador, inclusive entre estúdios diferentes.
- **FR-011**: Cliente, tatuador e estúdio MUST poder cancelar um pedido pendente enquanto ele
  estiver pendente e uma sessão confirmada até o seu início, sem prazo mínimo de antecedência;
  após o início, o cancelamento MUST ser recusado. Ao cancelar uma sessão confirmada, o
  intervalo MUST voltar a ficar livre.
- **FR-011a**: Todo agendamento MUST nascer pendente; apenas o tatuador do agendamento MUST
  poder aprová-lo (→ confirmado, FR-008b) ou recusá-lo (→ recusado). Pedidos pendentes MUST
  NOT reservar vaga nem contar para macas, capacidade ou sobreposição: essas regras são
  verificadas na aprovação.
- **FR-011b**: Pedido pendente sem resposta MUST expirar automaticamente 48h após a
  solicitação ou no início da faixa pedida (data e faixa escolhidas pelo cliente, no fuso do
  estúdio), o que ocorrer primeiro (→ expirado); a expiração MUST notificar o cliente.
- **FR-011c**: O sistema MUST recusar um novo agendamento quando já existirem 2 agendamentos
  pendentes com o mesmo e-mail ou com o mesmo telefone do cliente (normalizados), somando
  todos os estúdios; vale para visitantes e clientes com conta. Agendamentos confirmados não
  contam para o limite.
- **FR-011d**: O cliente (com conta ou pelo link de visitante) MUST poder remarcar um
  pedido pendente (antes da data pedida) ou uma sessão confirmada (antes do início),
  escolhendo outra data e faixa do mesmo estúdio com o mesmo tatuador. A nova preferência
  passa pelas mesmas validações de um novo pedido (FR-007d, FR-007e, FR-011c); se falhar,
  nada muda. Remarcado, o agendamento MUST voltar a pendente com novo prazo de expiração
  (FR-011b), e o intervalo confirmado anterior, se houver, é liberado. Estúdio e tatuador não
  remarcam; podem cancelar.
- **FR-011e**: O tatuador do agendamento e o estúdio (dono ou membro) MUST poder marcar uma
  sessão confirmada como "não compareceu" a partir do início da sessão e até 7 dias depois;
  dentro desse prazo, a marcação MAY ser desfeita. Sessões confirmadas sem essa marcação MUST
  passar automaticamente a "concluída" 7 dias após o início. Marcações feitas pelo estúdio
  entram no registro de ações (FR-023).
- **FR-012**: O sistema MUST notificar as partes envolvidas sobre criação, aprovação, recusa,
  expiração, cancelamento e remarcação de agendamentos.
- **FR-012a**: O sistema MUST enviar ao cliente um lembrete por e-mail 24h antes do início de
  cada agendamento confirmado, com link para ver, remarcar ou cancelar; agendamentos
  confirmados quando já faltam menos de 24h não recebem lembrete (o e-mail de aprovação
  cumpre o papel). O sistema MUST enviar ao tatuador, às 18h (horário de Brasília) do dia
  anterior, um único e-mail com os agendamentos confirmados do dia seguinte em todos os
  estúdios, cada um com o horário no fuso do respectivo estúdio; dias sem agendamentos não
  geram e-mail. Cada lembrete MUST ser enviado no máximo uma vez.
- **FR-013**: O estúdio MUST visualizar todos os agendamentos do estúdio, filtráveis por
  tatuador; o tatuador MUST visualizar seus agendamentos de todos os estúdios.

**Portfólio**

- **FR-014**: O tatuador MUST poder publicar, editar e remover trabalhos (imagem, título,
  descrição opcional, estilos).
- **FR-015**: O portfólio MUST ser exibido publicamente em grade visual de imagens com
  visualização ampliada de cada trabalho.
- **FR-016**: A página do tatuador MUST listar os estúdios onde possui vínculo ativo, com
  acesso ao agendamento em cada um.

**Descoberta**

- **FR-017**: Visitantes MUST poder buscar estúdios e tatuadores por nome, cidade e estilo.
  O estilo de um estúdio MUST ser derivado dos tatuadores com vínculo ativo nele (o estúdio
  corresponde ao estilo X se ao menos um desses tatuadores trabalha nesse estilo).

**Privacidade**

- **FR-018**: Nome, e-mail e telefone do cliente MUST ser visíveis apenas ao estúdio e ao
  tatuador do agendamento (e ao próprio cliente); nunca em páginas ou buscas públicas.
- **FR-019**: Os dados de contato do cliente de um agendamento, a descrição e as imagens de
  referência MUST ser anonimizados (imagens apagadas) 12 meses após o fim do agendamento;
  data, estúdio, tatuador e status são mantidos para histórico.
- **FR-020**: Qualquer usuário MUST poder excluir a própria conta. A exclusão remove o perfil
  de tatuador (e o portfólio), exclui o estúdio de que é dono (FR-020a), remove a conta das
  equipes de que é membro, encerra os vínculos, cancela os agendamentos futuros em que é
  parte com notificação às demais partes e anonimiza os dados pessoais da conta.
- **FR-020a**: O dono MUST poder excluir o estúdio sem excluir a própria conta. A exclusão do
  estúdio encerra os vínculos, cancela os agendamentos futuros do estúdio com notificação,
  remove todos os membros da equipe (sem excluir as contas deles) e tira o estúdio de
  páginas e buscas.

**Equipe do estúdio**

- **FR-021**: O dono do estúdio MUST poder convidar membros por e-mail (link válido por 7
  dias; quem já tem conta aceita entrando nela, quem não tem cria a conta pelo link), listar
  os membros e remover qualquer um a qualquer momento. A remoção MUST tirar imediatamente o
  acesso do membro àquele estúdio, sem afetar a conta dele nem o que ele faz em outros
  estúdios ou como tatuador. O dono não pode ser membro do próprio estúdio.
- **FR-022**: Membros MUST poder fazer tudo o que o dono faz na gestão do dia a dia: agenda
  (padrão semanal, exceções, avulsos, dias de atendimento dos tatuadores, janela de
  antecedência), vínculos (convidar, cancelar convite, encerrar) e agendamentos (ver
  detalhes e contato, cancelar). Somente o dono MUST poder editar o perfil público e o número
  de macas, gerenciar membros e excluir o estúdio.
- **FR-023**: Toda ação de gestão feita por dono ou membro (alterações de agenda, de vínculos,
  de dias de atendimento, de janela de antecedência e cancelamentos de agendamento) MUST ser
  registrada com autor, data/hora e descrição do que mudou. O dono MUST poder consultar esse
  registro, filtrando por membro e por período. O registro MUST ser mantido por 12 meses.
- **FR-024**: Notificações por e-mail destinadas ao estúdio MUST ir para o e-mail do dono;
  notificações in-app do estúdio MUST ser visíveis para o dono e todos os membros.

### Key Entities

- **Estúdio**: local de atendimento; tem exatamente um dono (conta) e uma equipe de membros; nome, endereço, contato, descrição, número de macas
  (limite de atendimentos simultâneos) e janela de antecedência para agendamento (mínima e
  máxima). Possui faixas de atendimento e vínculos com
  tatuadores.
- **Tatuador**: profissional; nome, bio, estilos. Possui portfólio e vínculos com estúdios.
- **Vínculo**: relação estúdio↔tatuador; status (convidado, ativo, recusado, expirado,
  cancelado, encerrado) e dias da semana em que o tatuador atende no estúdio.
- **Faixa de Atendimento** (formerly referred to as "Horário Disponível"): intervalo de
  data/hora em que o estúdio atende, com capacidade (sessões simultâneas, ≤ macas). Clientes
  escolhem data e faixa como preferência; o tatuador encaixa a sessão nas faixas da data.
- **Padrão Semanal**: regra recorrente do estúdio (dias da semana, faixas de horário,
  capacidade) que gera Faixas de Atendimento; exceções por data bloqueiam ou alteram as
  faixas da data.
- **Agendamento**: pedido de sessão; referencia estúdio, tatuador, cliente, data e faixa
  preferidas e status (pendente, confirmado, recusado, expirado, cancelado, concluído, não
  compareceu).
  Quando confirmado, tem início e duração definidos pelo tatuador. Contém os detalhes do
  pedido: descrição, local do corpo, tamanho e até 3 imagens de referência. Só sessões
  confirmadas ocupam macas e capacidade.
- **Trabalho de Portfólio**: imagem publicada pelo tatuador com título, descrição e estilos.
- **Conta**: pessoa usuária; nome, e-mail, telefone, senha. Pode ter perfil de Tatuador, ser
  dona de até 1 estúdio, membro de vários e agendar como cliente.
- **Membro do Estúdio**: relação conta↔estúdio que dá acesso à gestão do estúdio em nome do
  dono; uma conta pode ser membro de vários estúdios (e também tatuador vinculado a eles).
- **Registro de Ação**: entrada do histórico de gestão do estúdio; autor (dono ou membro),
  data/hora, tipo de ação e descrição da mudança.
- **Cliente**: pessoa que solicita o agendamento; nome, e-mail e telefone. Pode ser
  visitante (sem conta) ou qualquer conta; identificado pelo e-mail.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Um cliente envia um pedido de sessão (escolha de estúdio, tatuador, data e
  faixa, com detalhes) em menos de 3 minutos.
- **SC-002**: Um estúdio configura sua agenda semanal e vincula um tatuador em menos de 10
  minutos no primeiro uso.
- **SC-003**: 0% de sessões confirmadas sobrepostas para um mesmo tatuador, acima das macas
  do estúdio ou sem estúdio/tatuador.
- **SC-004**: O portfólio de um tatuador com até 100 trabalhos é exibido em menos de 2
  segundos para o visitante.
- **SC-005**: 90% dos usuários concluem o primeiro agendamento sem ajuda.

## Assumptions

- Pagamentos, sinal (depósito) e orçamento estão fora do escopo da v1.
- Notificações são enviadas por e-mail e dentro da aplicação; SMS/WhatsApp ficam fora da v1.
- Duração da sessão é definida pelo tatuador na aprovação, em múltiplos de 30 min (FR-008b).
- Todos os horários usam o fuso horário do estúdio.
- Aplicação web responsiva (uso em celular e desktop); apps nativos fora da v1.
- Avaliações/reviews, chat entre as partes e moderação de conteúdo ficam fora da v1.
- Backup automatizado e plano de recuperação de desastre ficam fora da v1; em caso de falha
  do servidor, dados (agendamentos, contas, imagens) podem ser perdidos.
- A equipe do estúdio tem só dois papéis na v1 (dono e membro); permissões granulares por
  membro ficam fora da v1.
- Autenticação por e-mail e senha, com recuperação de senha por e-mail (FR-003a).
- E-mails são enviados pela API do Gmail a partir de uma conta pessoal @gmail.com no MVP (o
  remetente é esse endereço, não um domínio próprio). O limite diário de envio de contas
  pessoais (cerca de 500 destinatários por dia) é aceito no MVP; acima dele, os e-mails são
  adiados até a liberação da cota, o que pode atrasar a chegada de pedidos e lembretes. A
  migração para domínio próprio (Google Workspace ou provedor transacional) fica para depois
  do MVP, quando o volume exigir.
- O MVP é dimensionado para até ~3 mil agendamentos por mês (~80–150 por dia), limite
  decorrente da cota de e-mail da conta Gmail pessoal com ~6 e-mails por sessão confirmada.
  Acima disso, a migração de envio de e-mail passa a ser necessária.
