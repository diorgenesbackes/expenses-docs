# Documentação do Expenses

Este diretório reúne requisitos, arquitetura, especificações de funcionalidades,
modelagem do banco e guidelines de desenvolvimento. As instruções para executar o projeto ficam no
[guia de infraestrutura local](../expenses-infrastructure/README.md).

## Índice e ordem de leitura

| Documento | Finalidade |
| --- | --- |
| [PRD](PRD.md) | Entender o produto, os requisitos e os critérios de aceite. |
| [HLD](HLD.md) | Entender a arquitetura e as decisões técnicas vigentes. |
| [FDD de criação de usuário e autenticação](fdd/FDD-Criacao-Usuario-Autenticacao/FDD-Criacao-Usuario-Autenticacao.md) | Consultar a especificação detalhada da funcionalidade de identidade. |
| [Plano de desenvolvimento de identidade](fdd/FDD-Criacao-Usuario-Autenticacao/1-planejamento.md) | Proposta de entregas, decisões pendentes, dependências e validação do cadastro e autenticação. |
| [Desenvolvimento de identidade](fdd/FDD-Criacao-Usuario-Autenticacao/2-desenvolvimento.md) | Histórico consolidado de cadastro E2 e login/sessões E3/E4, contratos e limitações. |
| [Testes da FDD de identidade](fdd/FDD-Criacao-Usuario-Autenticacao/3-testes.md) | Ciclos de testes, evidências históricas e situação do Definition of Done. |
| [Operação do cadastro](fdd/FDD-Criacao-Usuario-Autenticacao/4-operacao.md) | Implantação, reversão e investigação de resultados incertos. |
| [Medição local de cadastro](fdd/FDD-Criacao-Usuario-Autenticacao/3-testes-latencia-cadastro.json) | Vinte amostras HTTP com Cognito real; meta ainda não atendida. |
| [Levantamento para modelagem](db/DB-Modelagem.md) | Consultar os requisitos e as propostas originais para o banco. |
| [Modelo físico — sprints 1 a 3](db/DB-Modelo-Fisico.md) | Entender o modelo implementado e a relação com os scripts SQL. |
| [Padrão de testes e Definition of Done](testing/README.md) | Cobertura obrigatória, ambiente descartável, critérios de conclusão e formato dos registros. |
| [Plano de testes — local primeiro](testing/2026-10-08_12-02-50/1-plano.md) | Inventário dos 98 casos e proposta de projeto separado para integração, API, navegador, carga e segurança. |
| [Execução dos testes locais](testing/2026-10-08_12-02-50/2-execucao.md) | Resultados efetivos, evidências, alertas e pendências do ciclo histórico. |
| [Guidelines de desenvolvimento](guidelines/README.md) | Consultar princípios comuns, padrões de backend, frontend e Liquibase, checklists e fontes pesquisadas. |
| [Contratos de APIs síncronas](guidelines/API-SINCRONA.md) | Padronizar requests, responses, headers, paginação, erros e idempotência. |
| [Observabilidade](guidelines/OBSERVABILIDADE.md) | Padronizar logs, correlação, métricas, tracing, dashboards e alertas. |
| [Git Flow](guidelines/GIT-FLOW.md) | Definir branches, promoção seletiva, commits, PRs, releases e hotfixes a partir do documento-base fornecido. |

O levantamento registra o estado e as propostas da etapa em que foi produzido;
o modelo físico descreve a implementação das sprints 1 a 3. A documentação de requisitos
não significa que todas as funcionalidades já estejam implementadas. Para verificar
o comportamento disponível, consulte o código e os READMEs dos projetos.

## Organização

- `PRD.md` e `HLD.md`: documentos gerais do produto e da arquitetura.
- `fdd/`: um diretório `FDD-<Funcionalidade>/` para cada FDD, com especificação e histórico numerado.
- `db/`: documentos de banco, com nomes `DB-<Assunto>.md`.
- `testing/`: somente o padrão estrutural `README.md` na raiz e pastas de ciclos nomeadas por data-hora, cada uma com `1-plano.md` e `2-execucao.md`.
- `guidelines/`: regras de desenvolvimento e revisão, exemplos e referências, com distinção entre práticas atuais e adoção futura.

Ao adicionar uma FDD, inclua sua especificação neste índice. Os documentos auxiliares devem ser acessíveis pela navegação da própria FDD; cada novo ciclo de testes deve ser registrado em seu `3-testes.md`. Os links são relativos ao arquivo que os contém; ao mover ou renomear arquivos, atualize também suas referências.

### Padrão de histórico por FDD

```text
fdd/
  FDD-<Funcionalidade>/
    FDD-<Funcionalidade>.md
    1-planejamento.md
    2-desenvolvimento.md
    3-testes.md
    4-operacao.md
```

| Arquivo | Responsabilidade |
| --- | --- |
| `FDD-<Funcionalidade>.md` | Especificação e critérios de aceite vigentes, com versão e decisões de produto. |
| `1-planejamento.md` | Etapas, dependências, decisões de implementação e critérios previstos. |
| `2-desenvolvimento.md` | Histórico por entrega, com datas, contratos implementados, decisões, alterações e pendências. |
| `3-testes.md` | Índice dos ciclos de testes e evidências, com links para `testing/` e situação real do DoD. |
| `4-operacao.md` | Implantação, configuração, recuperação, reversão e diagnóstico quando aplicáveis. |

Usar esses nomes para todas as FDDs; criar cada documento na etapa correspondente ou sinalizá-lo como não iniciado. Conteúdo adicional deve seguir `<numero>-<etapa>-<assunto>.<extensao>`, associado a uma etapa existente. Por exemplo, `3-testes-latencia-cadastro.json` identifica uma evidência histórica de testes. Novas etapas posteriores usam o próximo numeral, sem renumerar etapas já referenciadas.

Não criar novos arquivos soltos para cada incremento do mesmo desenvolvimento: acrescentar seções datadas ao documento da etapa, preservando decisões, falhas e resultados anteriores. O documento de testes da FDD referencia os planos/execuções centrais, sem duplicá-los. A especificação vigente não deve ser confundida com o comportamento de uma entrega histórica.

### Padrão de ciclos de testes

```text
testing/
  README.md
  AAAA-MM-DD_HH-mm-ss/
    1-plano.md
    2-execucao.md
```

Usar o fuso `America/Sao_Paulo` e identificar o objetivo/FDD dentro dos arquivos. Cada ciclo tem preparação, cenários, critérios, resultado e limpeza rastreáveis. Casos automatizados individuais ficam na matriz de cenários; uma pasta pode reunir a campanha que os executa. As regras completas e os modelos de seções estão no [padrão estrutural](testing/README.md).

## Implementação e referências

- [Backend e testes](../expenses-service/README.md).
- [Frontend](../expenses-ui/README.md).
- [Liquibase, migrations e validação do schema](../expenses-liquibase/README.md).
- [POC e materiais do legado](../extras/README.md).
- [Análise e diagramas C4 de identidade](../extras/c4/criacao-usuario-autenticacao-c4.md).

Para visão geral e acesso rápido, consulte o [README da raiz](../README.md).
