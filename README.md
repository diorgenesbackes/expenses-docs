# Documentação do Expenses

Este diretório reúne requisitos, arquitetura, especificações de funcionalidades e
modelagem do banco. As instruções para executar o projeto ficam no
[guia de infraestrutura local](../expenses-infrastructure/README.md).

## Índice e ordem de leitura

| Documento | Finalidade |
| --- | --- |
| [PRD](PRD.md) | Entender o produto, os requisitos e os critérios de aceite. |
| [HLD](HLD.md) | Entender a arquitetura e as decisões técnicas vigentes. |
| [FDD de criação de usuário e autenticação](fdd/FDD-Criacao-Usuario-Autenticacao.md) | Consultar a especificação detalhada da funcionalidade de identidade. |
| [Levantamento para modelagem](db/DB-Modelagem.md) | Consultar os requisitos e as propostas originais para o banco. |
| [Modelo físico — sprint-1](db/DB-Modelo-Fisico.md) | Entender o modelo implementado e a relação com os scripts SQL. |

O levantamento registra o estado e as propostas da etapa em que foi produzido;
o modelo físico descreve a implementação da sprint-1. A documentação de requisitos
não significa que todas as funcionalidades já estejam implementadas. Para verificar
o comportamento disponível, consulte o código e os READMEs dos projetos.

## Organização

- `PRD.md` e `HLD.md`: documentos gerais do produto e da arquitetura.
- `fdd/`: especificações de funcionalidades, com nomes `FDD-<Funcionalidade>.md`.
- `db/`: documentos de banco, com nomes `DB-<Assunto>.md`.

Ao adicionar um documento, inclua-o neste índice. Os links são relativos ao arquivo
que os contém; ao mover ou renomear arquivos, atualize também suas referências.

## Implementação e referências

- [Backend e testes](../expenses-service/README.md).
- [Frontend](../expenses-ui/README.md).
- [Liquibase, migrations e validação do schema](../expenses-liquibase/README.md).
- [POC e materiais do legado](../extras/README.md).
- [Análise e diagramas C4 de identidade](../extras/c4/criacao-usuario-autenticacao-c4.md).

Para visão geral e acesso rápido, consulte o [README da raiz](../README.md).
