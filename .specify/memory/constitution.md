<!--
Sync Impact Report
- Version change: (template) → 1.0.0
- Modified principles: placeholders → definidos
  - [PRINCIPLE_1_NAME] → I. Qualidade de Código
  - [PRINCIPLE_2_NAME] → II. Código Enxuto
  - [PRINCIPLE_3_NAME] → III. SOLID
  - [PRINCIPLE_4_NAME] → IV. Clean Code
  - [PRINCIPLE_5_NAME] → V. YAGNI
- Added principles: VI. Commits Semânticos (Convenção Angular)
- Added sections: Padrões de Qualidade; Fluxo de Desenvolvimento
- Removed sections: nenhuma
- Templates:
  - ✅ .specify/templates/plan-template.md (Constitution Check é preenchido dinamicamente; sem mudança)
  - ✅ .specify/templates/spec-template.md (sem restrições novas; sem mudança)
  - ✅ .specify/templates/tasks-template.md (nota de commit alinhada à convenção Angular)
  - ✅ .claude/skills/speckit-*/SKILL.md (sem referências desatualizadas)
- Follow-up TODOs: nenhum
-->

# Tatuagem Constitution

## Core Principles

### I. Qualidade de Código

Todo código entregue MUST ser legível, testável e livre de avisos do linter/formatter adotado
pelo projeto. Código que não compila, não passa no lint ou quebra testes existentes MUST NOT
ser integrado. Duplicação de lógica MUST ser eliminada assim que surgir a segunda ocorrência
com a mesma razão de mudança.

**Rationale**: qualidade consistente reduz custo de manutenção e de onboarding.

### II. Código Enxuto

Cada mudança MUST conter apenas o código necessário para cumprir o requisito. Código morto,
comentado, imports não usados e dependências sem uso MUST ser removidos. Novas dependências
externas MUST ser justificadas no plano (o que resolvem e por que não é trivial implementar).
Funções SHOULD ter até ~20 linhas e arquivos SHOULD manter um único propósito.

**Rationale**: menos código significa menos bugs, menos superfície de ataque e revisão mais rápida.

### III. SOLID

O design MUST respeitar os princípios SOLID:

- **S**ingle Responsibility: cada módulo/classe/função tem uma única razão para mudar.
- **O**pen/Closed: estender comportamento sem modificar código estável quando houver
  variação real (não especulativa — ver Princípio V).
- **L**iskov Substitution: subtipos MUST ser substituíveis por seus tipos base sem quebrar
  contratos.
- **I**nterface Segregation: interfaces pequenas e específicas ao consumidor.
- **D**ependency Inversion: regras de negócio dependem de abstrações, não de detalhes de
  infraestrutura (banco, HTTP, frameworks).

**Rationale**: baixo acoplamento e alta coesão tornam o sistema testável e evolutivo.

### IV. Clean Code

- Nomes MUST revelar intenção; abreviações obscuras e nomes genéricos (`data`, `tmp`,
  `manager`) MUST NOT ser usados sem contexto.
- Funções MUST fazer uma única coisa, em um único nível de abstração, com no máximo 3
  parâmetros (acima disso, agrupar em objeto).
- Efeitos colaterais MUST ser explícitos; erros MUST ser tratados, nunca silenciados.
- Comentários explicam o *porquê*, não o *o quê*; código autoexplicativo é preferido.
- Formatação MUST ser automatizada pelo formatter do projeto.

**Rationale**: código é lido muito mais vezes do que escrito.

### V. YAGNI (You Aren't Gonna Need It)

Funcionalidades, abstrações, configurações e pontos de extensão MUST NOT ser implementados
antes de existir um requisito concreto na especificação. Generalização só é permitida a partir
da segunda necessidade real. Qualquer complexidade adicional MUST ser registrada na tabela
"Complexity Tracking" do plano com justificativa.

**Rationale**: código especulativo custa manutenção e raramente acerta a necessidade futura.

### VI. Commits Semânticos (Convenção Angular)

Todo commit MUST seguir a convenção Angular:

```text
<type>(<scope>): <subject>

<body>

<footer>
```

- `type` MUST ser um de: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`,
  `build`, `ci`, `chore`, `revert`.
- `scope` é opcional e indica o módulo afetado.
- `subject` MUST estar no imperativo, sem ponto final, com até 72 caracteres.
- Mudanças incompatíveis MUST conter `BREAKING CHANGE:` no footer; issues referenciadas
  no footer (ex.: `Closes #12`).
- Cada commit MUST ser atômico: uma mudança lógica, deixando o build verde.
- Versões do produto seguem SemVer derivado dos commits (`feat` → MINOR, `fix`/`perf` →
  PATCH, `BREAKING CHANGE` → MAJOR).

**Rationale**: histórico legível, changelog automatizável e versionamento previsível.

## Padrões de Qualidade

- Lint e formatter MUST rodar localmente (pre-commit) e no CI; o CI MUST bloquear merges
  com falhas.
- Mensagens de commit MUST ser validadas automaticamente (ex.: commitlint com
  `@commitlint/config-conventional` ou equivalente).
- Toda lógica de negócio MUST ter testes automatizados; correções de bug MUST incluir teste
  que reproduza o defeito.
- Segredos e credenciais MUST NOT ser versionados.

## Fluxo de Desenvolvimento

- Trabalho acontece em branches de feature; a branch principal MUST permanecer estável.
- Toda mudança MUST passar por revisão de código verificando conformidade com esta
  constituição (SOLID, Clean Code, YAGNI, código enxuto e mensagem de commit).
- Refatorações MUST ir em commits `refactor` separados de mudanças de comportamento.
- Pull requests SHOULD ser pequenos e focados em uma única história ou correção.

## Governance

Esta constituição prevalece sobre quaisquer outras práticas do projeto. Emendas MUST ser
propostas via pull request com commit `docs(constitution): ...`, descrevendo a motivação e o
impacto nos templates dependentes.

Versionamento da constituição segue SemVer:

- **MAJOR**: remoção ou redefinição incompatível de princípios ou regras de governança.
- **MINOR**: novo princípio/seção ou expansão material de orientações.
- **PATCH**: clarificações, redação e correções sem mudança semântica.

Todo plano (`/speckit-plan`) MUST executar o "Constitution Check" contra estes princípios;
violações MUST ser justificadas em "Complexity Tracking" ou corrigidas antes da implementação.
Revisões de código MUST verificar conformidade.

**Version**: 1.0.0 | **Ratified**: 2026-11-12 | **Last Amended**: 2026-11-12
