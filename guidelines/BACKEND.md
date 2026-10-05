# Guideline de backend

[Índice](README.md) · [Princípios comuns](PRINCIPIOS.md) · [Contratos HTTP](API-SINCRONA.md) · [Observabilidade](OBSERVABILIDADE.md) · [Execução e testes](../../expenses-service/README.md)

Aplicação: C#/.NET 10, ASP.NET Core, EF Core 10 e Npgsql/PostgreSQL 17. As regras abaixo orientam novas funcionalidades; os endpoints atuais são diagnósticos e não representam a implementação completa do produto.

## 1. Camadas e módulos

| Projeto | Responsabilidade | Dependências permitidas |
| --- | --- | --- |
| `Expenses.Domain` | Entidades, valores, invariantes e contratos de domínio. | Biblioteca base do .NET; sem ASP.NET Core, EF Core ou SDKs externos. |
| `Expenses.Application` | Casos de uso, coordenação, validação de entrada, portas e resultados de aplicação. | Domain. |
| `Expenses.Infrastructure` | EF Core, consultas, persistência e adaptadores de serviços externos. | Domain; Application quando implementar uma porta definida nela. |
| `Expenses.Api` | Contratos HTTP, controllers, autenticação, tradução de erros e composição. | Application; Infrastructure no registro/configuração de dependências. |
| `Expenses.Test` | Testes de comportamento, contratos HTTP e integração. | Projetos necessários ao cenário. |

Atualmente Infrastructure referencia apenas Domain. Uma futura referência a Application só deve ser introduzida para implementar seus contratos, sem criar dependência inversa. Application e Domain NÃO DEVEM conhecer a infraestrutura. Essa direção segue a [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html).

Organizar funcionalidades por Identity, Household, Finance e Income dentro das camadas quando houver código que justifique a divisão. Não criar um projeto por tabela nem interfaces vazias para antecipar funcionalidades. Comunicação entre módulos usa contratos internos, não HTTP para a própria aplicação. O [HLD](../HLD.md) permite transações locais que atravessem módulos.

**Ajuste incremental de contratos:** os DTOs e o `UserMapper` existentes estão em Domain. Para novos casos, colocar requests/responses específicos de HTTP na API e entradas/resultados independentes de HTTP em Application. Manter em Domain apenas modelos com significado de domínio. Migrar os existentes em alteração dedicada, preservando JSON, validação e Swagger. Evitar DTOs duplicados quando não há uma fronteira real a proteger.

## 2. Fluxo de um caso de uso

O controller DEVE obter entrada, identidade e cancelamento, chamar o caso de uso e produzir a resposta. Application coordena autorização, leitura, regras, transação e resultado. Domain concentra invariantes relevantes. Infrastructure executa as operações externas.

Uma consulta simples pode usar uma entidade com propriedades e um serviço pequeno. Para fechamento, rateio e reabertura, explicitar métodos e políticas que impeçam estados inválidos. Não colocar o algoritmo financeiro no controller, no mapper ou em uma consulta SQL oculta.

**Validação:** formato e limites na fronteira; autorização e pré-condições no caso de uso; invariantes de negócio no domínio; integridade persistente nas constraints. São responsabilidades complementares. O caso de uso não deve depender exclusivamente da validação MVC, pois pode ter outro consumidor.

Mapeamentos simples DEVEM ser explícitos e testáveis. Uma biblioteca de mapeamento só é justificável quando reduzir trabalho real sem esconder conversões monetárias, normalização ou campos sensíveis.

### Convenções C# para novas implementações

- Manter nullable reference types habilitados; tratar ausência de forma explícita, sem espalhar `!` para silenciar o compilador.
- Usar PascalCase para tipos/membros públicos, camelCase para parâmetros/variáveis e sufixo `Async` para métodos assíncronos, preservando convenções do projeto.
- Preferir resultados/DTOs imutáveis quando apropriado; entidades com invariantes devem restringir mutações que permitam estados inválidos.
- Evitar classes base de serviço/controller que acumulem dependências; usar composição e métodos com finalidade explícita.
- Exceções inesperadas seguem o tratamento central; resultados esperados de domínio precisam de representação clara, sem um framework genérico de resultados obrigatório.
- Manter código assíncrono legível, descarte explícito dos recursos criados pelo próprio código e comentários que expliquem decisões.

## 3. Injeção de dependência e tempos de vida

| Lifetime | Uso indicado | Custo e cuidado |
| --- | --- | --- |
| Transient | Objeto leve sem estado, criado a cada resolução. | Pode gerar alocações repetidas; não pressupor uma instância única por request. |
| Scoped | Caso de uso, repositório e `DbContext` da requisição. | Compartilha estado dentro do escopo; não capturar em singleton. |
| Singleton | Serviço compartilhado, imutável ou seguro para concorrência. | Vive até o encerramento; não guardar usuário, casa ou estado de request. |

Usar injeção por construtor e deixar o container descartar os objetos que criou. Não chamar `BuildServiceProvider` durante o registro nem resolver dependências por Service Locator em serviços de negócio. As [diretrizes de DI do .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/guidelines) fundamentam essas restrições.

O registro atual de `IUserService`, `IUserRepository` e `ExpensesDbContext` como scoped é o padrão para serviços que compartilham essa unidade de trabalho. A resolução tardia no `DatabaseHealthCheck` é uma exceção técnica deliberada: ocorre no escopo do runner para converter falhas de configuração em estado unhealthy. Não copiar esse padrão para casos de uso.

## 4. EF Core e acesso ao PostgreSQL

### Mapeamento e unidade de trabalho

**DEVE:** manter `IEntityTypeConfiguration<T>` na Infrastructure e mapear explicitamente schema, tabela, coluna, tamanho, precisão, nulabilidade e campos gerados. O mapeamento descreve o schema existente; não o cria. A evolução é exclusiva do Liquibase. NÃO usar `EnsureCreated`, `Database.Migrate`, migrations EF ou seed automático no startup.

`DbContext` tem duração curta e não suporta operações simultâneas na mesma instância. Aguardar cada operação antes de iniciar outra; não executar contagem e listagem em `Task.WhenAll` com o mesmo contexto. Criar contextos separados apenas quando houver necessidade e sem presumir que compartilharão transação ou snapshot. Referência: [configuração e lifetime de DbContext](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/).

### Consultas

- DEVE haver filtro de autorização/escopo aplicável, limite e ordenação determinística antes da materialização.
- RECOMENDADO usar `AsNoTracking` para leitura e projeção dos campos necessários no banco. Entidades rastreadas são apropriadas para alterações controladas.
- Evitar N+1, carregamento indiscriminado de relacionamentos e consultas por item de uma coleção.
- Não devolver `IQueryable` pelo contrato do caso de uso nem permitir que o controller componha consultas EF.
- Parametrizar SQL; nomes dinâmicos de colunas/ordenação precisam de lista permitida, pois parâmetros de valor não os tornam seguros.
- Medir SQL, quantidade de consultas e plano antes de otimizar. Queries compiladas e pooling de contexto não são requisitos iniciais.

A [documentação de consultas eficientes do EF Core](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying) sustenta projeção, limites, escolha de paginação e análise de índices.

O diagnóstico atual usa `OrderBy(Id)`, `Skip/Take`, página inicial 1, tamanho padrão 20 e máximo 100. A contagem e os itens são consultas sequenciais sem snapshot comum: mudanças concorrentes podem produzir diferença entre eles. Preservar esse contrato; para grandes volumes, CONSIDERAR cursor com chave de desempate única, definindo um novo contrato. Não prometer consistência de snapshot onde ela não existe.

### Escritas e transações

Uma chamada a `SaveChanges` é transacional quando o provider oferece suporte. Se o caso de uso precisar proteger leituras, múltiplas gravações ou locks, delimitar uma transação explícita. Repositórios participantes não devem confirmar pedaços independentes de um mesmo caso de uso. `DbContext` já fornece a base da unidade de trabalho; uma interface adicional é opcional. Referência: [transações no EF Core](https://learn.microsoft.com/en-us/ef/core/saving/transactions).

Manter transações curtas. Não manter locks enquanto espera interação do usuário ou chama o Cognito. A transação PostgreSQL não inclui o fornecedor externo; seguir a compensação e reconciliação especificadas no [FDD de identidade](../fdd/FDD-Criacao-Usuario-Autenticacao.md).

## 5. Concorrência, idempotência e histórico financeiro

Novas escritas mensais DEVEM seguir o [protocolo do modelo físico](../db/DB-Modelo-Fisico.md):

1. Obter o ator autenticado e verificar casa, vínculo, papel e estado.
2. Abrir a transação e adquirir locks na ordem **casa → período**, revalidando condições que podem mudar.
3. Comparar `edit_version`, dono/token da concessão e validade do bloqueio de edição; aplicar o prazo definido de cinco minutos.
4. Gravar alteração, revisão, auditoria e resultado de idempotência pertinentes; incrementar a versão atomicamente.
5. Confirmar a transação e responder. Versão/concessão incompatível deve produzir conflito, sem sobrescrita silenciosa.

Uma coluna `edit_version` não basta: o mapeamento ou a atualização condicional DEVE comparar a versão esperada. Configurar o token EF e tratar `DbUpdateConcurrencyException`, ou usar comando condicional e verificar linhas afetadas. Referência: [concorrência otimista no EF Core](https://learn.microsoft.com/en-us/ef/core/saving/concurrency).

Os triggers atuais não autorizam o ator nem incrementam sua versão por ele. Fechamento exige snapshot consistente e o par versão/rateio conforme as FKs adiadas. Reabertura preserva versões antigas; consultar histórico não executa novamente o algoritmo atual.

Para comandos com idempotência prevista no modelo, escopar a chave por operação/casa/ator, comparar o fingerprint do conteúdo, coordenar disputa por unicidade e persistir resultado seguro junto da operação. Mesma chave com conteúdo diferente não é repetição válida. Testar requisição repetida e perda de resposta após commit. Desabilitar um botão não resolve duplicação no servidor.

## 6. Dinheiro, datas e identificadores

- **Dinheiro:** `decimal` no .NET e `numeric(18,2)` no modelo atual. Validar limites e casas decimais antes de persistir; o banco pode arredondar valores com escala excedente. Não usar `double` em cálculo financeiro.
- **Arredondamento:** documentar modo, etapa e distribuição de resíduos; não escolher silenciosamente `ToEven` ou `AwayFromZero`. A política definitiva do rateio ainda precisa ser ratificada. Testar totais, centavos restantes, zero e limites.
- **Datas:** instantes em UTC; competência é `DateOnly`/`date`, sem conversão para meia-noite em timezone arbitrário. `timestamptz` não preserva o nome do fuso original; Npgsql exige atenção ao envio de UTC. Ver [tipos de data/hora do Npgsql](https://www.npgsql.org/doc/types/datetime.html).
- **IDs:** preservar UUID no contrato atual. O formato de IDs públicos com prefixo mencionado nos documentos precisa de decisão específica antes da adoção.
- **JSON:** definir representação financeira que preserve centavos no JavaScript; `decimal` no servidor não garante precisão após `JSON.parse`. Strings decimais são a proposta para contratos financeiros novos, sujeita a ADR e alinhamento com o frontend. Valores `bigint` também precisam de limite seguro ou representação textual.

Usar um relógio substituível, como `TimeProvider`, ao implementar expiração/concessão para testar limites sem esperas reais. Não alterar o instante de referência no meio de uma mesma validação de prazo sem necessidade.

## 7. Async, cancelamento e resiliência

Propagar `CancellationToken` do request a serviços e chamadas de I/O. NÃO usar `.Result`, `.Wait()`, `async void` ou `Task.Run` para disfarçar I/O bloqueante. O negócio continua síncrono do ponto de vista HTTP. Essas práticas seguem as [recomendações de desempenho do ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices?view=aspnetcore-10.0).

Definir orçamento por operação e distinguir cancelamento do cliente, timeout interno e falha da dependência. Os diagnósticos atuais usam cinco segundos; isso não define automaticamente o limite de todos os casos de uso. Timeout de conexão e de comando têm papéis distintos.

Retry precisa de falha transitória identificável, limite de tentativas/tempo e segurança de repetição. Não habilitar retry genérico em escritas ou chamadas Cognito. Se houver transação explícita com uma estratégia de retry, verificar como repetir a unidade inteira com o provider. Cancelamento ou perda de conexão após commit pode deixar o resultado desconhecido para o cliente; usar idempotência/consulta de resultado, sem afirmar que nada foi gravado.

## 8. HTTP, erros e documentação

Usar `/v1` para negócio; preservar `/health` e `/health/db` como diagnósticos. GET não altera estado de negócio. Documentar requests, defaults, limites, respostas e exemplos no OpenAPI.

O [padrão de APIs síncronas](API-SINCRONA.md) é a referência para formato de request/response, headers, paginação, status e catálogo de erros. Novos contratos usam objetos de negócio em sucesso, paginação `{ items, page, pageSize, totalCount }` e `204` sem corpo quando não houver representação. As exceções do FDD, como `422`/`423` em Identity, ficam explícitas nesse padrão.

Preservar contratos específicos já existentes. Exemplo: `/health/db` retorna apenas `{"status":"up"}` ou `{"status":"down"}`; não precisa ser convertido para o formato de erro de negócio.

Para erros da API, usar `ProblemDetails`/`ValidationProblemDetails`, com `code`, `traceId` W3C e `correlationId` no contrato alvo. O handler atual ainda usa `HttpContext.TraceIdentifier` em `traceId`; migrar essa associação e uniformizar erros MVC faz parte da adoção. Não expor stack trace, SQL, connection string, tokens ou payload do fornecedor. O cliente não deve depender do texto traduzido para decidir o fluxo. Referência: [RFC 9457 — Problem Details](https://www.rfc-editor.org/rfc/rfc9457.html).

## 9. Segurança e operação

Autenticação ainda é uma etapa futura. `UseAuthorization()` isoladamente não a implementa. Endpoints protegidos só podem ser publicados após configurar o esquema, validar a sessão e aplicar autorização por recurso. O identificador da casa recebido do cliente precisa ser confrontado com o vínculo do ator; uma FK composta protege integridade, não acesso. Ver [autorização baseada em recurso no ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased?view=aspnetcore-10.0).

Seguir Cognito intermediado pelo backend e cookies seguros conforme o FDD. Projetar CSRF e CORS junto dos domínios reais de implantação; não liberar origem arbitrária com credenciais. Segredos locais ficam em User Secrets/arquivos ignorados; produção usa configuração segura da infraestrutura. O usuário `admin` local não deve se tornar a identidade operacional de produção: separar migração DDL e aplicação DML por privilégio mínimo.

Logs DEVEM ser estruturados e incluir resultado, duração e correlação, sem segredos ou dados pessoais desnecessários. Não habilitar logging de dados sensíveis do EF em ambiente compartilhado. Instrumentação OpenTelemetry está prevista no HLD; sua presença na documentação não comprova configuração atual.

Aplicar o [padrão de observabilidade](OBSERVABILIDADE.md): eventos JSON catalogados, separação de `traceId`/`requestId`/`correlationId`, métricas nativas e `expenses.*`, durações em segundos nos histogramas e labels sem IDs de usuários/casas. Validar diagnósticos de exceções tratadas no .NET 10 e nome lógico do pool Npgsql antes da exportação.

`/health` verifica resposta da aplicação. `/health/db` verifica acesso ao banco, não schema, autorização DML de cada tabela ou capacidade de fechar um mês. Não usar falha do banco como razão automática para reiniciar continuamente a API. Swagger e listagem de usuários permanecem em Development.

## 10. Testes e checklist

| Nível | O que verificar |
| --- | --- |
| Unidade | Regras de cálculo/estado, validação, mapeamento, limites e expiração; dependências externas substituídas por portas. |
| HTTP | Contrato, status, validação, ambiente, autenticação/autorização quando implementadas e erros sanitizados, com `WebApplicationFactory`. |
| PostgreSQL real | SQL traduzido, precisão, constraints, campos gerados, transações, concorrência e isolamento entre casas. |

Não usar EF InMemory ou SQLite como prova de comportamento específico do PostgreSQL. Não simular `DbSet` para testar a tradução SQL. A ferramenta oficial para hospedar a API em testes está descrita em [testes de integração do ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-10.0).

Comandos existentes, a partir de `expenses-service/`:

```bash
dotnet build Expenses.sln
dotnet test Expenses.sln --no-build
python3 scripts/test-postgres.py
```

O último comando exige Docker Desktop/PostgreSQL e Liquibase; cria e remove um banco isolado. Sem essa execução, testes PostgreSQL podem estar ignorados: não declarar integração completa apenas porque a suíte sem banco passou.

- [ ] Controller sem regra financeira ou consulta EF direta.
- [ ] Dependências e lifetimes coerentes; cancelamento propagado.
- [ ] Contrato documentado, dados mínimos expostos e acesso autorizado.
- [ ] Consulta limitada; escrita com integridade, transação e concorrência adequadas.
- [ ] Valores, datas e snapshots preservados corretamente.
- [ ] Cenários relevantes de sucesso, limite, falha e disputa testados.
- [ ] Alteração de schema entregue pelo Liquibase e compatível com o mapeamento.
- [ ] Contrato HTTP e telemetria seguem os guias transversais, com correlação e dados sensíveis verificados.
