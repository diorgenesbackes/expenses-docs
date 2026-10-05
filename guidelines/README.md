# Guidelines de desenvolvimento do Expenses

Versão inicial: **2026-10-05**. Base: pesquisa em documentação oficial e análise do código e dos documentos deste repositório nessa data.

Estes guias definem como desenvolver e revisar o Expenses. As regras propostas passam a orientar novas mudanças; sua publicação não significa que todas já estejam implementadas ou automatizadas. O ambiente local continua documentado no [guia de infraestrutura](../../expenses-infrastructure/README.md).

## Ordem de leitura

| Documento | Conteúdo |
| --- | --- |
| [Princípios comuns](PRINCIPIOS.md) | Clean Architecture, Clean Code, SOLID, escolha de padrões, revisão e decisões arquiteturais. |
| [Git Flow](GIT-FLOW.md) | Branches, subtarefas, promoção seletiva, PRs, commits, releases, hotfixes e integração de migrations. |
| [Contratos de APIs síncronas](API-SINCRONA.md) | Requests, responses, paginação, headers, erros, concorrência, idempotência e compatibilidade. |
| [Observabilidade](OBSERVABILIDADE.md) | Schema de logs, correlação, catálogo de métricas, tracing, SLIs, dashboards e alertas. |
| [Backend](BACKEND.md) | C#, ASP.NET Core, DI, EF Core/Npgsql, contratos, segurança, concorrência e testes. |
| [Frontend](FRONTEND.md) | React, TypeScript, Vite, organização, estado, integração, acessibilidade e testes. |
| [Liquibase e PostgreSQL](LIQUIBASE.md) | Changelogs YAML, SQL, integridade, índices, evolução, rollback e validação. |
| [Pesquisa e referências](PESQUISA-E-REFERENCIAS.md) | Fontes primárias, critérios de pesquisa e limites das recomendações. |

## Como interpretar as regras

- **DEVE / NÃO DEVE:** regra do projeto para novas implementações. Uma exceção exige justificativa explícita na revisão e, se alterar a arquitetura, registro de decisão.
- **RECOMENDADO:** escolha padrão; outra solução é aceitável quando o contexto e os custos estiverem demonstrados.
- **CONSIDERAR:** alternativa condicionada a uma necessidade; não determina a instalação de uma biblioteca ou criação de uma abstração.
- **Atual:** observado no código ou nos scripts existentes.
- **A adotar / proposto:** orientação ainda dependente de implementação e validação.

Os exemplos são ilustrativos. Não adicionam endpoints, tabelas, bibliotecas ou configurações ao sistema.

## Base técnica verificada

| Área | Base do repositório |
| --- | --- |
| Backend | .NET/ASP.NET Core 10; EF Core 10.0.12; provider Npgsql EF Core 10.0.3; Swashbuckle 10.2.3. |
| Testes backend | xUnit 2.9.3, `WebApplicationFactory`, teste isolado com PostgreSQL e Liquibase. |
| Frontend | React 19.2, TypeScript 6.0, Vite 8.3, Oxlint; Node 24 no build Docker. |
| Banco | PostgreSQL 17; ambiente registrado como validado com 17.11. |
| Migrações | Liquibase 4.33.0, changelogs YAML e arquivos SQL separados dos rollbacks. |
| Ambiente | Docker Desktop; Compose em `expenses-infrastructure/`; Nginx publica o frontend e encaminha `/api/`. |

As versões vêm dos manifests e dos READMEs; intervalos de versão não equivalem a um inventário de dependências instalado. Usar o lockfile do frontend e revisar compatibilidade nas atualizações. Não adotar APIs de outra versão nem funcionalidades licenciadas do Liquibase por aparecerem na documentação mais recente.

## Decisões transversais

O [HLD vigente](../HLD.md) define um **monólito modular**, com Identity, Household, Finance e Income. As operações de negócio terminam na requisição HTTP; chamadas internas são métodos no mesmo processo. `async/await` para I/O é compatível com essa decisão. Filas, jobs de negócio, microsserviços, eventos distribuídos e comunicação em tempo real não fazem parte do MVP.

O banco é único, organizado em schemas. O Liquibase controla sua evolução; o EF Core acessa e mapeia dados. Valores financeiros e autorizações são decididos pelo backend; a interface auxilia a entrada e a apresentação. Snapshots de fechamentos preservam o resultado histórico.

PRD/FDD definem comportamento de produto; o HLD define a arquitetura vigente; o modelo físico e os scripts descrevem o banco implementado. Um conflito entre essas fontes deve gerar uma decisão documentada, não uma escolha silenciosa. Referências antigas a microsserviços não substituem o HLD atual. Estes guidelines não alteram requisitos de produto.

## Situação atual e adoção incremental

| Tema | Atual | Próximo critério de adoção |
| --- | --- | --- |
| Separação backend | API, Application, Domain e Infrastructure; DTOs HTTP e mapper ainda no Domain. | Novos contratos específicos de HTTP na API; modelos de casos de uso em Application, sem mudança incompatível dos existentes. |
| Acesso ao banco | EF Core scoped, leitura sem tracking, paginação limitada e cancelamento. | Reproduzir esses cuidados nas consultas novas; escritas exigirão transação, autorização e controle de concorrência. |
| Autenticação | Integração Cognito e sessão descritas nos documentos, ainda não implementadas. | Implementar autenticação, autorização por recurso e CSRF antes de publicar dados de negócio protegidos. |
| Diagnóstico de usuários | Disponível apenas em Development. | Manter a restrição; qualquer publicação exige contrato e política de acesso próprios. |
| Contratos HTTP | Paginação e Problem Details existentes; formatos do FDD ainda exigem compatibilização. | Aplicar o catálogo de erros, padronizar headers e manter as exceções documentadas no guia de APIs. |
| Observabilidade | Logs pontuais e health checks; `traceId` do handler usa o ID local do request. | Correlacionar Activity/headers/logs, configurar JSON e exportação, depois validar métricas, dashboards e alertas. |
| Frontend | Template React; sem funcionalidades de negócio. | Organizar por funcionalidades à medida que forem criadas. |
| TypeScript | `strict` ausente nos tsconfigs da aplicação e das ferramentas. | Habilitar tipagem estrita em mudança dedicada e corrigir os erros encontrados. |
| Testes frontend | Sem runner ou suíte configurados. | Selecionar e configurar ferramentas junto da primeira funcionalidade testável. |
| Proxy local | `/api/` disponível no Nginx; Vite sem proxy. | Configurar o proxy do Vite ao implementar a primeira integração com a API. |
| Migrações | Oito changesets da sprint-1 e validação isolada. | Preservar o histórico e adaptar as verificações de contagem/rollback quando novas migrações forem adicionadas. |
| Automação | Scripts locais de build, lint e testes. | Integrar os checks aplicáveis à CI; este trabalho não criou um pipeline. |
| Git Flow | Nesta análise, a pasta ainda não é um repositório Git inicializado. | Adotar `main`/`hlg`/`dev`, promoção seletiva por história, proteções e rastreabilidade conforme o guia; configurar o remoto em tarefa própria. |

## Fluxo de uma mudança

1. Identificar requisito, módulo responsável, dados envolvidos e história/tarefa no fluxo Git.
2. Definir contrato, erros, permissões e compatibilidade antes de implementar.
3. Consultar o guia da área e os pontos de integração com as demais.
4. Implementar a menor mudança completa, incluindo migração e testes quando necessários.
5. Executar os checks aplicáveis e registrar o que passou, falhou ou foi ignorado.
6. Atualizar documentação de comportamento e registrar decisões que afetem outras funcionalidades.

Promoção entre dev, homologação e produção segue o [Git Flow do projeto](GIT-FLOW.md), com evidências vinculadas ao conteúdo aprovado.

Não exigir testes novos para uma simples correção textual ou mudança sem comportamento. Para regras financeiras, isolamento entre casas, autenticação e migrações, os cenários de falha fazem parte da entrega.

## Manutenção

Atualizar estes documentos quando houver mudança de versão principal, arquitetura, contrato transversal ou descoberta operacional relevante. Uma regra nova deve incluir motivação e forma de verificação. Evitar repetir comandos operacionais extensos: os READMEs de cada projeto continuam sendo a referência de execução.
