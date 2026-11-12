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

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Estúdio configura agenda e vincula tatuadores (Priority: P1)

O responsável pelo estúdio cria o perfil do estúdio, define os horários disponíveis para
atendimento e vincula os tatuadores que atendem ali.

**Why this priority**: sem estúdio, vínculo e horários não existe agenda; é a base de todo o
fluxo de agendamento.

**Independent Test**: criar um estúdio, convidar um tatuador, ter o convite aceito e publicar
horários com capacidade; verificar que os horários aparecem no estúdio e podem ser reservados
com o tatuador vinculado até o limite de vagas.

**Acceptance Scenarios**:

1. **Given** um estúdio cadastrado, **When** o estúdio convida um tatuador cadastrado,
   **Then** o tatuador recebe o convite e o vínculo só fica ativo após o aceite.
2. **Given** um estúdio com tatuador vinculado, **When** o estúdio cadastra horários
   disponíveis informando a capacidade (número de atendimentos simultâneos), **Then** os
   horários ficam visíveis para agendamento com qualquer tatuador vinculado enquanto houver
   vaga.
3. **Given** um tatuador vinculado, **When** o estúdio desfaz o vínculo, **Then** o tatuador
   deixa de aparecer como opção naquele estúdio e novos agendamentos com ele ali ficam
   bloqueados.
4. **Given** um estúdio com padrão semanal configurado, **When** ele bloqueia uma data
   (ex.: feriado) ou cria um horário avulso, **Then** apenas aquela data é afetada e o padrão
   continua valendo para as demais semanas.

---

### User Story 2 - Cliente agenda sessão escolhendo estúdio e tatuador (Priority: P1)

Um cliente escolhe um estúdio e um tatuador vinculado a ele (ou um tatuador e um estúdio
onde ele atende), seleciona um horário disponível e solicita o agendamento.

**Why this priority**: é o valor central do produto — transformar disponibilidade em sessão
marcada.

**Independent Test**: com um estúdio, um tatuador vinculado e horários publicados, um cliente
conclui um agendamento; tentar agendar sem estúdio ou sem tatuador deve ser impossível.

**Acceptance Scenarios**:

1. **Given** um estúdio com tatuadores vinculados e horários livres, **When** o cliente
   seleciona estúdio, tatuador e horário, **Then** o agendamento é registrado como
   pendente, a vaga fica reservada e o tatuador é notificado para aprovar ou recusar.
2. **Given** o cliente partindo do perfil de um tatuador, **When** ele inicia um agendamento,
   **Then** o sistema exige a escolha de um dos estúdios onde o tatuador está vinculado antes
   de exibir horários.
3. **Given** o cliente partindo do perfil de um estúdio, **When** ele inicia um agendamento,
   **Then** o sistema exige a escolha de um tatuador vinculado antes de exibir horários.
4. **Given** um horário já ocupado, **When** outro cliente tenta reservá-lo para o mesmo
   tatuador, **Then** o sistema impede a reserva.
5. **Given** um agendamento existente, **When** cliente, tatuador ou estúdio o cancela,
   **Then** o horário volta a ficar disponível e as demais partes são notificadas.
6. **Given** um agendamento pendente, **When** o tatuador aprova, **Then** o agendamento
   passa a confirmado e o cliente e o estúdio são notificados; **When** o tatuador recusa,
   **Then** o agendamento passa a recusado, a vaga é liberada e o cliente é notificado.

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

---

### Edge Cases

- Tatuador vinculado a dois estúdios não pode ter dois agendamentos sobrepostos, mesmo em
  estúdios diferentes.
- Desvínculo de um tatuador com agendamentos futuros naquele estúdio: os agendamentos futuros
  são mantidos até serem cancelados explicitamente pelo estúdio ou tatuador; novos são
  bloqueados.
- Estúdio remove um horário que já possui agendamento: o sistema impede a remoção até o
  agendamento ser cancelado.
- Dois clientes tentam reservar ao mesmo tempo a última vaga de um horário, ou o mesmo
  tatuador no mesmo horário: apenas um é concluído; o outro recebe aviso de
  indisponibilidade.
- Tatuador sem nenhum vínculo ativo: perfil e portfólio continuam visíveis, mas o botão de
  agendar indica que não há estúdios disponíveis.
- Agendamento em horário passado não é permitido.
- Tatuador não responde a um pedido pendente: o pedido expira em 48h ou no início do
  horário (o que vier antes), a vaga é liberada e o cliente é notificado.
- Estúdio sem tatuadores vinculados não aceita agendamentos.
- Horário com todas as vagas ocupadas deixa de ser exibido como disponível.
- Estúdio altera o padrão semanal ou bloqueia uma data com agendamentos ativos: as
  ocorrências com agendamentos são mantidas até que estes sejam cancelados; as demais seguem
  a nova regra.
- Estúdio reduz a capacidade de um horário abaixo do número de agendamentos ativos nele: o
  sistema impede a alteração até que agendamentos sejam cancelados.
- Ao escolher um horário, só são oferecidos tatuadores vinculados que não tenham outro
  agendamento sobreposto (em qualquer estúdio).

## Requirements *(mandatory)*

### Functional Requirements

**Contas e perfis**

- **FR-001**: O sistema MUST permitir cadastro com os perfis Estúdio e Tatuador, cada um com
  página pública (nome, descrição, foto, cidade/endereço para estúdio, estilos para tatuador).
- **FR-002**: O sistema MUST permitir que clientes solicitem agendamentos como visitante,
  informando nome, e-mail e telefone; o visitante MUST poder acompanhar e cancelar o
  agendamento por um link de acesso enviado ao e-mail informado.
- **FR-002a**: O cliente MAY criar uma conta com perfil Cliente; com conta, MUST visualizar
  todos os seus agendamentos em uma área própria. Agendamentos feitos como visitante com o
  mesmo e-mail MUST ser associados à conta quando ela for criada.
- **FR-003**: Cada conta MUST possuir um único perfil (Estúdio, Tatuador ou Cliente) na v1.

**Vínculos**

- **FR-004**: O estúdio MUST poder convidar tatuadores cadastrados; o vínculo MUST ficar ativo
  apenas após aceite do tatuador.
- **FR-005**: Estúdio e tatuador MUST poder encerrar um vínculo a qualquer momento.
- **FR-006**: Um tatuador MUST poder ter vínculos ativos com vários estúdios, e um estúdio com
  vários tatuadores.

**Agenda**

- **FR-007**: Apenas o estúdio MUST poder criar, editar e remover horários disponíveis. Cada
  horário é do estúdio (não de um tatuador) e MUST ter uma capacidade ≥ 1 de atendimentos
  simultâneos.
- **FR-007a**: Um horário MUST aceitar agendamentos até atingir sua capacidade; cada
  agendamento no mesmo horário MUST ter um tatuador diferente.
- **FR-007b**: O estúdio MUST poder definir um padrão semanal recorrente (dias da semana,
  faixas de horário e capacidade) que gera os horários disponíveis, e MUST poder registrar
  exceções: bloquear ou editar ocorrências em datas específicas e criar horários avulsos.
- **FR-008**: Todo agendamento MUST referenciar exatamente um estúdio, um tatuador com vínculo
  ativo nesse estúdio e um horário disponibilizado por esse estúdio.
- **FR-009**: O sistema MUST impedir agendamentos sem estúdio ou sem tatuador selecionado.
- **FR-010**: O sistema MUST impedir sobreposição de agendamentos para o mesmo tatuador,
  inclusive entre estúdios diferentes.
- **FR-011**: Cliente, tatuador e estúdio MUST poder cancelar um agendamento; o horário MUST
  voltar a ficar disponível.
- **FR-011a**: Todo agendamento MUST nascer pendente; apenas o tatuador do agendamento MUST
  poder aprová-lo (→ confirmado) ou recusá-lo (→ recusado). Agendamentos pendentes MUST
  ocupar vaga e contar para a regra de sobreposição (FR-010); recusa MUST liberar a vaga.
- **FR-011b**: Agendamento pendente sem resposta MUST expirar automaticamente 48h após a
  solicitação ou no início do horário, o que ocorrer primeiro (→ expirado); a expiração MUST
  liberar a vaga e notificar o cliente.
- **FR-012**: O sistema MUST notificar as partes envolvidas sobre criação, aprovação, recusa,
  expiração e cancelamento de agendamentos.
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

### Key Entities

- **Estúdio**: local de atendimento; nome, endereço, contato, descrição. Possui horários
  disponíveis e vínculos com tatuadores.
- **Tatuador**: profissional; nome, bio, estilos. Possui portfólio e vínculos com estúdios.
- **Vínculo**: relação estúdio↔tatuador; status (convidado, ativo, recusado, encerrado).
- **Horário Disponível**: intervalo de data/hora oferecido por um estúdio para atendimento,
  com capacidade (vagas simultâneas, ex.: macas). Vagas livres = capacidade − agendamentos
  ativos no horário.
- **Padrão Semanal**: regra recorrente do estúdio (dias da semana, faixas de horário,
  capacidade) que gera Horários Disponíveis; exceções por data bloqueiam ou alteram
  ocorrências.
- **Agendamento**: reserva de um horário; referencia estúdio, tatuador, cliente e status
  (pendente, confirmado, recusado, expirado, cancelado, concluído). Agendamentos ativos = pendentes +
  confirmados.
- **Trabalho de Portfólio**: imagem publicada pelo tatuador com título, descrição e estilos.
- **Cliente**: pessoa que solicita o agendamento; nome, e-mail e telefone. Pode ser
  visitante (sem conta) ou ter conta com perfil Cliente; identificado pelo e-mail.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Um cliente conclui um agendamento (escolha de estúdio, tatuador e horário) em
  menos de 3 minutos.
- **SC-002**: Um estúdio configura sua agenda semanal e vincula um tatuador em menos de 10
  minutos no primeiro uso.
- **SC-003**: 0% de agendamentos sobrepostos para um mesmo tatuador ou sem estúdio/tatuador.
- **SC-004**: O portfólio de um tatuador com até 100 trabalhos é exibido em menos de 2
  segundos para o visitante.
- **SC-005**: 90% dos usuários concluem o primeiro agendamento sem ajuda.

## Assumptions

- Pagamentos, sinal (depósito) e orçamento estão fora do escopo da v1.
- Notificações são enviadas por e-mail e dentro da aplicação; SMS/WhatsApp ficam fora da v1.
- Duração da sessão é definida pelo horário disponibilizado pelo estúdio.
- Todos os horários usam o fuso horário do estúdio.
- Aplicação web responsiva (uso em celular e desktop); apps nativos fora da v1.
- Avaliações/reviews, chat entre as partes e moderação de conteúdo ficam fora da v1.
- Autenticação por e-mail e senha, padrão de mercado.
