# Feature Specification: Agendamento para Tatuadores e Estúdios

**Feature Branch**: `001-agendamento-tatuagem`

**Created**: 2026-11-12

**Status**: Draft

**Input**: User description: "crie uma aplicação para agendamentos de tatuadores e estúdios de tatuagem. são dois perfis principais: tatuador e estúdio, e o estúdio pode configurar agenda, disponibilizando horários, e vincular tatuadores. já o tatuador pode estar vinculado a diversos estúdios e colocar seu portfolio à mostra estilo pinterest/behance. um estúdio pode ter vários tatuadores e um tatuador pode tatuar em vários estúdios também. porém, as agendas dependem inteiramente dos estúdios, ou seja, não é possível agendar com o tatuador sem selecionar um estúdio onde ele atende, e não é possível agendar em um estúdio sem selecionar um tatuador."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Estúdio configura agenda e vincula tatuadores (Priority: P1)

O responsável pelo estúdio cria o perfil do estúdio, define os horários disponíveis para
atendimento e vincula os tatuadores que atendem ali.

**Why this priority**: sem estúdio, vínculo e horários não existe agenda; é a base de todo o
fluxo de agendamento.

**Independent Test**: criar um estúdio, convidar um tatuador, ter o convite aceito e publicar
horários; verificar que os horários aparecem associados ao estúdio e ao tatuador vinculado.

**Acceptance Scenarios**:

1. **Given** um estúdio cadastrado, **When** o estúdio convida um tatuador cadastrado,
   **Then** o tatuador recebe o convite e o vínculo só fica ativo após o aceite.
2. **Given** um estúdio com tatuador vinculado, **When** o estúdio cadastra horários
   disponíveis, **Then** os horários ficam visíveis para agendamento
   [NEEDS CLARIFICATION: os horários são definidos por tatuador dentro do estúdio, ou são
   horários gerais do estúdio que qualquer tatuador vinculado pode ocupar?].
3. **Given** um tatuador vinculado, **When** o estúdio desfaz o vínculo, **Then** o tatuador
   deixa de aparecer como opção naquele estúdio e novos agendamentos com ele ali ficam
   bloqueados.

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
   seleciona estúdio, tatuador e horário, **Then** o agendamento é registrado
   [NEEDS CLARIFICATION: o agendamento é confirmado automaticamente ou fica pendente até
   aprovação do estúdio/tatuador?].
2. **Given** o cliente partindo do perfil de um tatuador, **When** ele inicia um agendamento,
   **Then** o sistema exige a escolha de um dos estúdios onde o tatuador está vinculado antes
   de exibir horários.
3. **Given** o cliente partindo do perfil de um estúdio, **When** ele inicia um agendamento,
   **Then** o sistema exige a escolha de um tatuador vinculado antes de exibir horários.
4. **Given** um horário já ocupado, **When** outro cliente tenta reservá-lo para o mesmo
   tatuador, **Then** o sistema impede a reserva.
5. **Given** um agendamento existente, **When** cliente, tatuador ou estúdio o cancela,
   **Then** o horário volta a ficar disponível e as demais partes são notificadas.

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
- Dois clientes tentam reservar o mesmo horário ao mesmo tempo: apenas um é concluído; o
  outro recebe aviso de indisponibilidade.
- Tatuador sem nenhum vínculo ativo: perfil e portfólio continuam visíveis, mas o botão de
  agendar indica que não há estúdios disponíveis.
- Agendamento em horário passado não é permitido.
- Estúdio sem tatuadores vinculados não aceita agendamentos.

## Requirements *(mandatory)*

### Functional Requirements

**Contas e perfis**

- **FR-001**: O sistema MUST permitir cadastro com os perfis Estúdio e Tatuador, cada um com
  página pública (nome, descrição, foto, cidade/endereço para estúdio, estilos para tatuador).
- **FR-002**: O sistema MUST permitir que clientes solicitem agendamentos
  [NEEDS CLARIFICATION: o cliente precisa criar conta (terceiro perfil) ou pode agendar como
  visitante informando nome e contato?].
- **FR-003**: Cada conta MUST possuir um único perfil (Estúdio ou Tatuador) na v1.

**Vínculos**

- **FR-004**: O estúdio MUST poder convidar tatuadores cadastrados; o vínculo MUST ficar ativo
  apenas após aceite do tatuador.
- **FR-005**: Estúdio e tatuador MUST poder encerrar um vínculo a qualquer momento.
- **FR-006**: Um tatuador MUST poder ter vínculos ativos com vários estúdios, e um estúdio com
  vários tatuadores.

**Agenda**

- **FR-007**: Apenas o estúdio MUST poder criar, editar e remover horários disponíveis.
- **FR-008**: Todo agendamento MUST referenciar exatamente um estúdio, um tatuador com vínculo
  ativo nesse estúdio e um horário disponibilizado por esse estúdio.
- **FR-009**: O sistema MUST impedir agendamentos sem estúdio ou sem tatuador selecionado.
- **FR-010**: O sistema MUST impedir sobreposição de agendamentos para o mesmo tatuador,
  inclusive entre estúdios diferentes.
- **FR-011**: Cliente, tatuador e estúdio MUST poder cancelar um agendamento; o horário MUST
  voltar a ficar disponível.
- **FR-012**: O sistema MUST notificar as partes envolvidas sobre criação, confirmação e
  cancelamento de agendamentos.
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
- **Horário Disponível**: intervalo de data/hora oferecido por um estúdio para atendimento.
- **Agendamento**: reserva de um horário; referencia estúdio, tatuador, cliente e status
  (ex.: pendente, confirmado, cancelado, concluído).
- **Trabalho de Portfólio**: imagem publicada pelo tatuador com título, descrição e estilos.
- **Cliente**: pessoa que solicita o agendamento; nome e contato.

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
