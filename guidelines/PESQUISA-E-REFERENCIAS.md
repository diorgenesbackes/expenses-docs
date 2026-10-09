# Pesquisa e referências dos guidelines

[Índice dos guidelines](README.md)

Consulta e análise: **2026-10-05**. Este documento registra a base da primeira versão dos guias; links também aparecem junto das recomendações que sustentam.

Complemento na mesma data: padrões específicos de [APIs síncronas](API-SINCRONA.md) e [observabilidade](OBSERVABILIDADE.md), confrontando os contratos do FDD com os endpoints, logs e configuração efetivamente existentes.

Complemento de Git Flow: adaptação do arquivo `03 - git-flow.md` fornecido pelo usuário. Uma [cópia de referência](referencias/git-flow-base.md), com quebras de linha normalizadas, preserva o conteúdo original; o [guia do projeto](GIT-FLOW.md) identifica e detalha o processo adotado. O arquivo original externo não foi alterado.

## Método e limites

1. Confrontar PRD, HLD, FDD e modelo físico com manifests, código, configurações públicas e scripts de teste.
2. Pesquisar fontes primárias: documentação dos mantenedores, normas técnicas e publicações dos autores dos princípios.
3. Verificar compatibilidade com .NET/EF Core 10, React 19, TypeScript 6, Vite 8, Liquibase OSS 4.33 e PostgreSQL 17.
4. Converter os fundamentos em regras revisáveis para o monólito modular do Expenses.
5. Separar comportamento existente, regra para código novo e proposta dependente de implementação.

“Melhor prática” não significa regra universal. Escolhas como pastas por funcionalidade, colocação de DTOs, representação monetária e ferramentas de teste exigem contexto. Os guias explicitam essas escolhas; a documentação de um framework não as torna obrigatórias por si só.

A análise não é benchmark, auditoria de segurança, certificação de acessibilidade ou comprovação de uma implantação em produção. Não houve alteração de runtime nem execução de migração para produzir estes documentos. Exemplos foram escritos para explicar os padrões, não extraídos como implementações prontas para o produto.

## Evidências do repositório

| Evidência | Decisão orientada por ela |
| --- | --- |
| [HLD](../HLD.md) e [FDD de identidade](../fdd/FDD-Criacao-Usuario-Autenticacao/FDD-Criacao-Usuario-Autenticacao.md) | Monólito síncrono, módulos internos, Cognito pelo backend e sessão por cookies. |
| [Modelo físico](../db/DB-Modelo-Fisico.md) e [changelog da sprint-1](../../expenses-liquibase/changelogs/sprint-1/changelog.yaml) | Precisão financeira, FKs compostas, snapshots, concorrência e histórico de migrações. |
| [Program.cs](../../expenses-service/Expenses.Api/Program.cs) e [UserRepository.cs](../../expenses-service/Expenses.Infrastructure/Repository/UserRepository.cs) | DI scoped, endpoints de diagnóstico, EF Core e paginação implementada. |
| [package.json](../../expenses-ui/package.json) e [tsconfig.app.json](../../expenses-ui/tsconfig.app.json) | Ferramentas presentes e lacunas de tipagem/testes, sem presumir adoção futura. |
| [README backend](../../expenses-service/README.md), [README Liquibase](../../expenses-liquibase/README.md) e [README frontend](../../expenses-ui/README.md) | Comandos reais, pré-requisitos e limites do ambiente local. |

## Arquitetura e qualidade

| Fonte primária | Contribuição |
| --- | --- |
| [Robert C. Martin — The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) | Direção das dependências e isolamento das políticas. |
| [Robert C. Martin — Solid Relevance](https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html) | Aplicação dos princípios SOLID além de uma estrutura específica de classes. |
| [Microsoft — Common Web Application Architectures](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures) | Organização de aplicações .NET e composição de adaptadores. |
| [Martin Fowler — YAGNI](https://martinfowler.com/bliki/Yagni.html) | Evitar capacidades especulativas e abstrações sem necessidade. |
| [Fowler — Repository](https://martinfowler.com/eaaCatalog/repository.html) e [Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html) | Vocabulário e responsabilidades dos padrões de persistência. |
| [Google — What to Look for in a Code Review](https://google.github.io/eng-practices/review/reviewer/looking-for.html) | Revisão de desenho, complexidade, comportamento e clareza. |

## Git Flow e entrega

| Fonte | Contribuição |
| --- | --- |
| [Documento-base fornecido](referencias/git-flow-base.md) | Branches `main`/`hlg`/`dev`, histórias criadas de main, tasks, promoção seletiva e fix para os três destinos. |
| [Git — merge](https://git-scm.com/docs/git-merge) | Ancestralidade, merge commits e resolução controlada de conflitos. |
| [Git — revert](https://git-scm.com/docs/git-revert) | Reversão por novo commit e efeitos sobre reintegração de merges. |
| [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) | Estrutura de commits e indicação de incompatibilidade. |
| [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html) | Critérios para versões imutáveis de entrega. |
| [GitHub — Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) | Referência para configurar revisão, checks e restrições; o provedor de hospedagem ainda não foi definido. |

A promoção seletiva vem do documento-base. Merge commits, sincronização a partir de main, validação isolada do candidato e rastreabilidade de artefatos são complementações para tornar esse processo consistente com o monorepo e o Liquibase. Não representam configuração remota já realizada.

## Backend e contratos HTTP

| Fonte primária | Contribuição |
| --- | --- |
| [Microsoft — DI Guidelines](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/guidelines) e [EF Core — DbContext](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/) | Escopos, descarte e limites de concorrência do contexto. |
| [ASP.NET Core — Best Practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices?view=aspnetcore-10.0) | I/O assíncrono e prevenção de bloqueios desnecessários. |
| [EF Core — Efficient Querying](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying) | Projeção, quantidade de dados e custo de consulta. |
| [EF Core — Transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions) e [Concurrency](https://learn.microsoft.com/en-us/ef/core/saving/concurrency) | Atomicidade e detecção de atualização concorrente. |
| [Npgsql — Date and Time Handling](https://www.npgsql.org/doc/types/datetime.html) | Correspondência de instantes e datas com PostgreSQL. |
| [ASP.NET Core — Resource-based Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased?view=aspnetcore-10.0) | Autorização que depende do recurso carregado. |
| [ASP.NET Core — Integration Tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-10.0) | Hospedagem da API para testes HTTP. |
| [IETF — RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) | Estrutura de problemas HTTP com extensão controlada. |
| [IETF — RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) | Semântica dos métodos, status, conteúdo de resposta e headers HTTP. |

## Frontend, acessibilidade e segurança

| Fonte primária | Contribuição |
| --- | --- |
| [React — Purity](https://react.dev/reference/rules/components-and-hooks-must-be-pure) e [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks) | Renderização previsível e uso correto de hooks. |
| [React — State Structure](https://react.dev/learn/choosing-the-state-structure) | Evitar estado redundante ou contraditório. |
| [React — You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) e [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects) | Separação entre derivação, eventos e sincronização externa. |
| [TypeScript — strict](https://www.typescriptlang.org/tsconfig/strict.html), [noUncheckedIndexedAccess](https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html) e [exactOptionalPropertyTypes](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html) | Configuração explícita das garantias estáticas. |
| [MDN — Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) | Status HTTP, leitura de respostas e cancelamento no navegador. |
| [Vite — Env and Mode](https://vite.dev/guide/env-and-mode) | Variáveis públicas no bundle e configuração de build. |
| [Vitest](https://vitest.dev/guide/), [Testing Library](https://testing-library.com/docs/guiding-principles/) e [Playwright](https://playwright.dev/docs/best-practices) | Opções de testes por comportamento e fluxos de usuário. |
| [W3C — Forms](https://www.w3.org/WAI/tutorials/forms/) e [WCAG 2.2](https://www.w3.org/WAI/WCAG22/quickref/) | Semântica, campos, foco e critérios de acessibilidade. |
| [OWASP — Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) | Privilégio mínimo e autorização por acesso ao recurso. |
| [OWASP — Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) e [CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) | Proteção de sessão e requisições autenticadas por cookies. |

## Observabilidade

| Fonte primária | Contribuição |
| --- | --- |
| [W3C — Trace Context](https://www.w3.org/TR/trace-context/) | Contexto de tracing interoperável e significado dos identificadores. |
| [OpenTelemetry — Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/) | Mapeamento entre log estruturado, recurso e campos nativos de trace/span. |
| [ASP.NET Core — HTTP Logging](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-logging/?view=aspnetcore-10.0) e [OWASP — Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) | Captura limitada, sanitização e tratamento de dados sensíveis. |
| [ASP.NET Core — Metrics](https://learn.microsoft.com/en-us/aspnet/core/metrics/overview?view=aspnetcore-10.0) e [HTTP Metrics](https://learn.microsoft.com/en-us/aspnet/core/metrics/http?view=aspnetcore-10.0) | Instrumentos nativos, unidades e coleta. |
| [OpenTelemetry — HTTP Metrics](https://opentelemetry.io/docs/specs/semconv/http/http-metrics/) e [Database Metrics](https://opentelemetry.io/docs/specs/semconv/db/database-metrics/) | Convenções de instrumentos/atributos e estabilidade. |
| [OpenTelemetry .NET — Metrics Best Practices](https://opentelemetry.io/docs/languages/dotnet/metrics/best-practices/) | Ciclo de vida dos instrumentos e controle de cardinalidade. |
| [Npgsql — Metrics](https://www.npgsql.org/doc/diagnostics/metrics.html) | Nomes da versão 10 e identificação segura do pool. |
| [ASP.NET Core — Error Handling](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling?view=aspnetcore-10.0) | Supressão de diagnósticos de exceções tratadas no .NET 10. |
| [OpenTelemetry — Sampling](https://opentelemetry.io/docs/concepts/sampling/) | Diferença entre decisões na entrada e após o resultado do trace. |
| [Google SRE — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) | Alertas por consumo de orçamento de erro e múltiplas janelas. |
| [AWS — Logs Insights QL](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html) | Consulta dos eventos estruturados por correlação. |

O formato JSON, os códigos de erro, o namespace `expenses.*`, os eventos e os parâmetros iniciais de alertas são decisões propostas para o Expenses. Não são todos exigências das normas. Os alvos de disponibilidade/latência vêm do HLD; percentuais de sampling e janelas de alerta exigem validação operacional.

## Liquibase e PostgreSQL

| Fonte primária | Contribuição |
| --- | --- |
| [Liquibase 4.33 — Changeset](https://docs.liquibase.com/oss/user-guide-4-33/what-is-a-changeset) e [Checksum](https://docs.liquibase.com/oss/user-guide-4-33/what-is-a-changeset-checksum) | Identidade, ordem e detecção de mudança no histórico. |
| [Liquibase 4.33 — logicalFilePath](https://docs.liquibase.com/oss/reference-guide-4-33/changelog-attributes/logicalfilepath) | Preservação deliberada de identidade ao reorganizar caminhos. |
| [Liquibase 4.33 — sqlFile](https://docs.liquibase.com/oss/reference-guide-4-33/change-types/sqlfile) e [runInTransaction](https://docs.liquibase.com/oss/reference-guide-4-33/changelog-attributes/runintransaction) | Execução de SQL externo e unidade transacional. |
| [Liquibase 4.33 — Preconditions](https://docs.liquibase.com/oss/user-guide-4-33/what-are-preconditions) | Verificação das premissas de uma migração. |
| [PostgreSQL 17 — Constraints](https://www.postgresql.org/docs/17/ddl-constraints.html) e [Numeric Types](https://www.postgresql.org/docs/17/datatype-numeric.html) | Integridade, nulabilidade, precisão e escala. |
| [PostgreSQL 17 — Lexical Structure](https://www.postgresql.org/docs/17/sql-syntax-lexical.html) e [UUID Functions](https://www.postgresql.org/docs/17/functions-uuid.html) | Identificadores e geração nativa de UUID. |
| [PostgreSQL 17 — Multicolumn Indexes](https://www.postgresql.org/docs/17/indexes-multicolumn.html) e [Using EXPLAIN](https://www.postgresql.org/docs/17/using-explain.html) | Desenho de índice e investigação de custo. |
| [PostgreSQL 17 — CREATE INDEX](https://www.postgresql.org/docs/17/sql-createindex.html) e [ALTER TABLE](https://www.postgresql.org/docs/17/sql-altertable.html) | Operações concorrentes, locks e validação de constraints. |

Páginas genéricas do Liquibase podem redirecionar para versões e edições diferentes. As referências selecionadas usam OSS 4.33; menções a recursos Secure/Pro dentro delas não os tornam disponíveis no projeto. Os guias não exigem checks ou relatórios comerciais.

## Decisões que permanecem abertas

- Política final de arredondamento/rateio e formato monetário público.
- Domínios de implantação, detalhes dos cookies, renovação, CSRF e CORS.
- Adoção de testes frontend, roteamento e eventual cache de consultas.
- Migração incremental dos DTOs existentes, automação de checks e formalização das ADRs.
- Adoção do contrato de erros/correlação, exportadores, destino de métricas, retenção, sampling e alertas; revisão dos pontos de compatibilidade identificados no FDD.
- Inicialização/hospedagem Git, responsáveis, proteções, CI e promoção dos artefatos conforme o processo documentado.

Essas lacunas constam dos guias como trabalho futuro. Devem ser fechadas junto das funcionalidades correspondentes, com contratos e testes, sem confundir recomendação com implementação já entregue.
