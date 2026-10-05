# Padrão de logs, métricas e tracing

[Índice](README.md) · [Contratos HTTP](API-SINCRONA.md) · [Backend](BACKEND.md) · [HLD](../HLD.md)

**Status:** especificação para adoção. Atualmente há logs pontuais com `ILogger`, health checks e instrumentos disponibilizados pelas bibliotecas; não há configuração de exportação OpenTelemetry, dashboards ou alertas implementada neste trabalho. O HLD prevê CloudWatch para logs e OpenTelemetry/X-Ray para traces. O destino e a coleta de métricas precisam ser configurados na implantação.

## 1. Sinais e responsabilidades

| Sinal | Pergunta que responde | Regra |
| --- | --- | --- |
| Métrica | Quanto, com que frequência e em quanto tempo? | Dados agregados, dimensões limitadas e unidade explícita. |
| Log | O que ocorreu nesta execução? | Evento estruturado com correlação e informação segura. |
| Trace | Onde o tempo foi gasto e qual dependência falhou? | Spans conectados ao mesmo contexto, sem duplicar instrumentação. |
| Auditoria de negócio | Quem alterou um registro e o que deve ser preservado? | Persistência e retenção próprias; não substituir pelas métricas/logs operacionais. |

O backend continua sendo um processo modular: `service.name=expenses-api`; Identity/Household/Finance/Income são atributos de módulo, não serviços fictícios independentes. Eventos de telemetria não são eventos de negócio assíncronos nem introduzem filas no MVP.

## 2. Identidade e correlação

| Campo | Significado e origem |
| --- | --- |
| `traceId` | Trace W3C obtido de `Activity.Current.TraceId`, 32 hexadecimais; nunca substituir por `HttpContext.TraceIdentifier`. |
| `spanId` | Span ativo, 16 hexadecimais; pode mudar entre operações do mesmo trace. |
| `requestId` | Identificador local da requisição, obtido de `HttpContext.TraceIdentifier`. |
| `correlationId` | UUID validado/gerado conforme [o contrato HTTP](API-SINCRONA.md); devolvido em `X-Correlation-Id`. |

Propagar `traceparent`/`tracestate` por instrumentação compatível com [W3C Trace Context](https://www.w3.org/TR/trace-context/). Validar contexto externo e definir sua confiança no ingresso; IDs e flags enviados pelo cliente não autorizam acesso nem obrigam coleta ilimitada. Não colocar dados pessoais ou credenciais em baggage.

Manter esses identificadores num escopo de logging do request. Um comando CLI sem request pode ter trace próprio e `executionId`; omitir campos HTTP ausentes, sem inventar valores de usuário/casa. Traces não exportados por amostragem ainda podem ter IDs válidos nos logs.

O `traceId` atual do handler é, na realidade, o identificador local do request. Corrigir exige implementação e teste de correlação entre Problem Details, header, log e span. Não presumir que o nome do campo atual garante tracing distribuído.

## 3. Formato de logs

**Padrão:** eventos JSON, um por linha na saída de container, com campos pesquisáveis e tipos estáveis. O formato abaixo é o schema de consulta esperado no destino. O formatter/exporter/transformação deve produzi-lo: `ILogger` com console padrão não garante esse JSON automaticamente.

| Campo | Obrigatoriedade / conteúdo |
| --- | --- |
| `timestamp` | Sempre; instante UTC ISO 8601 com precisão de milissegundos. |
| `severity` | Sempre; `Trace`, `Debug`, `Information`, `Warning`, `Error` ou `Critical`. |
| `eventId`, `eventName` | Sempre; inteiro e nome estáveis do catálogo de eventos. |
| `message` | Sempre; texto do evento, sem payload/valores sensíveis interpolados. |
| `serviceName`, `serviceVersion`, `environment`, `instanceId` | Sempre em logs de serviço; aplicação, artefato imutável, ambiente e instância. |
| `module`, `operation` | Módulo/operação conhecidos; enums/catálogo, como `identity` e `identity.users.list`. |
| `traceId`, `spanId`, `correlationId`, `requestId` | Obrigatórios no fluxo HTTP instrumentado; omitir quando não existirem fora desse fluxo. |
| `httpMethod`, `route`, `statusCode`, `durationMs`, `outcome` | Evento terminal HTTP; rota é o template, status numérico quando enviado, duração numérica. |
| `errorCode`, `exceptionType`, `dependency` | Quando aplicáveis; valores classificados e seguros. |

Vocabulário de `outcome`: `success`, `rejected`, `failure`, `timeout`, `client_aborted` ou `idempotent_replay`, conforme a operação. Para HTTP, manter também o status real: `409` é rejeição esperada, não sucesso de negócio. Em request abortado sem resposta, omitir `statusCode`; não fabricar `499` como status enviado pela aplicação.

Exemplo **alvo**, com valores sintéticos:

```json
{
  "timestamp": "2026-10-05T14:30:00.125Z",
  "severity": "Information",
  "eventId": 1000,
  "eventName": "http.request.completed",
  "message": "HTTP request completed",
  "serviceName": "expenses-api",
  "serviceVersion": "commit-a1b2c3d",
  "environment": "development",
  "instanceId": "local-api-1",
  "module": "identity",
  "operation": "identity.users.list",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "correlationId": "97dfba56-bdf4-49ba-8f64-3cf9e14c134a",
  "requestId": "local-request-001",
  "httpMethod": "GET",
  "route": "/v1/identity/users",
  "statusCode": 200,
  "durationMs": 12.8,
  "outcome": "success"
}
```

No transporte OTLP, mapear `timestamp`, `severity` e `message` para Timestamp, SeverityText/SeverityNumber e Body; `traceId`/`spanId` vão para os campos nativos. Demais campos ficam em Resource/Attributes conforme a semântica. Não gerar um novo ID na transformação. Referência: [OpenTelemetry Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/).

Recursos OTel: `service.name`, `service.version`, `service.instance.id`, `deployment.environment.name`. No schema de logs, correspondem a `serviceName`, `serviceVersion`, `instanceId` e `environment`. No início, usar ambientes `development`, `test`, `staging`, `production`.

### Emissão e catálogo

Usar templates estáveis de `ILogger`, propriedades estruturadas e `EventId`; evitar concatenação/interpolação de mensagens e serialização de objetos inteiros. Medir duração com relógio monotônico (`Stopwatch`), não subtraindo relógios civis sujeitos a ajuste. Source-generated logging é uma opção para caminhos frequentes, não requisito para toda mensagem.

Catálogo inicial proposto:

| ID | Evento | Responsável |
| --- | --- | --- |
| 1000 | `http.request.completed` | Middleware de conclusão; uma vez por request concluído. |
| 1001 | `http.request.aborted` | Middleware; substitui o terminal de conclusão quando o cliente aborta. |
| 1002 | `http.request.failed` | Fronteira de erro; uma ocorrência por falha técnica final. |
| 2000 | `dependency.call.failed` | Adaptador; detalhe necessário de falha externa, sem repetir stack no handler. |
| 3000 | `identity.login.rejected` | Caso de uso Identity, após classificar a causa permitida. |
| 3001 | `identity.reconciliation.required` | Divergência persistida que requer reconciliação. |
| 5000 | `finance.period.conflict` | Caso de uso Finance. |
| 5001 | `finance.period.closed` | Caso de uso, após commit bem-sucedido. |
| 7000 | `database.migration.completed` | Processo controlado de migração, com resultado. |

Reservar 4000–4999 para Household e 6000–6999 para Income. Novos eventos entram no catálogo sem reutilizar IDs antigos com outro significado. O middleware registra resumo; a fronteira que trata a falha registra o diagnóstico. Não repetir a mesma exceção no repository, service, controller e handler.

### Níveis e volume

| Nível | Política |
| --- | --- |
| Trace/Debug | Detalhe técnico temporário e seguro; desabilitado normalmente em produção. |
| Information | Resumo de request, transições relevantes e rejeições esperadas. O resumo HTTP usa esse nível mesmo quando o status é 5xx; o diagnóstico tem seu próprio evento Error. |
| Warning | Degradação recuperada, conflito operacional incomum ou repetição anormal; não usar para todo `400`. |
| Error | Falha técnica final que impede a operação, registrada uma vez com contexto seguro. |
| Critical | Processo incapaz de operar ou risco concreto à integridade; requer ação imediata. |

Consultas de health bem-sucedidas podem ter logs reduzidos/excluídos para evitar ruído; manter suas métricas e registrar transições/falhas com limitação de repetição. Na adoção inicial, manter um resumo por request de negócio. Redução futura por amostragem deve preservar falhas e explicitar a política; não calcular SLIs a partir de logs amostrados.

### Dados permitidos

Não registrar bodies, query strings, URLs com IDs, SQL com parâmetros, connection strings, senhas, cookies, `Authorization`, chave de idempotência, token de concessão, valores financeiros ou resposta Cognito por padrão. Usar allowlist de headers/atributos e escapar/limitar texto externo para impedir injeção de linhas. Desabilitar captura automática desses dados também nos spans e exportadores. Referências: [OWASP Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) e [HTTP logging/redaction no ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-logging/?view=aspnetcore-10.0).

Se a investigação exigir vínculo com ator/casa, usar pseudônimo por HMAC com segredo controlado, finalidade e retenção definidas; não hash simples de e-mail/IP nem ID bruto por padrão. `userAgent` bruto e PII do FDD não são campos obrigatórios de todo evento. Mensagem/stack de exceção só em canal restrito após sanitização validada; o logger atual corretamente evita mensagens que podem conter segredos.

Auditoria persistente mantém seu controle de acesso e ciclo de vida. Retenção de logs deve ser finita e configurada por ambiente; duração e acesso precisam de decisão operacional antes da publicação, sem inventar prazo legal neste guia.

## 4. Métricas: nomes, unidades e cardinalidade

Reutilizar métricas nativas antes de criar equivalentes. Novas métricas de aplicação usam namespace `expenses.*`, nomes em minúsculas com pontos, unidade UCUM e tipo documentados. Histogramas de duração usam **segundos**; logs usam **milissegundos** em `durationMs`. Quantidades monetárias não são valores de telemetria operacional.

O exporter pode traduzir nomes para underscores e acrescentar `_total`/`_seconds`; documentar essa tradução no dashboard. Não criar dois instrumentos para o mesmo evento por diferença de nomenclatura. Reutilizar `Meter`/instrumentos de vida longa e registrar uma medição no ponto semântico correto. Referências: [métricas do ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/metrics/overview?view=aspnetcore-10.0) e [boas práticas OTel .NET](https://opentelemetry.io/docs/languages/dotnet/metrics/best-practices/).

### Catálogo técnico mínimo

| Instrumento | Tipo / unidade | Fonte e finalidade |
| --- | --- | --- |
| `http.server.request.duration` | Histogram / `s` | ASP.NET Core: duração, volume pelo count e distribuição de status. |
| `http.server.active_requests` | UpDownCounter / `{request}` | ASP.NET Core: requests simultâneos; não pressupor que oferece dimensão de rota. |
| `db.client.operation.duration` | Histogram / `s` | Npgsql 10: custo de comandos de banco. |
| `db.client.connection.count` | UpDownCounter / `{connection}` | Npgsql 10: conexões por estado `idle`/`used`. |
| `db.client.connection.max` | UpDownCounter / `{connection}` | Npgsql 10: capacidade máxima do pool para comparação com uso. |
| `expenses.dependency.duration` | Histogram / `s` | Novo: chamadas lógicas externas não cobertas adequadamente, como Cognito; dimensões `dependency`, `operation`, `outcome`. |
| `expenses.operation.duration` | Histogram / `s` | Novo: caso de uso completo; `module`, `operation`, `outcome`. Não somar seu count ao count HTTP como se fossem requests distintos. |

Confirmar tipo/exportação dos instrumentos na versão efetivamente instalada antes de construir painéis. Fontes: [HTTP metrics do ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/metrics/http?view=aspnetcore-10.0), [convenções HTTP OTel](https://opentelemetry.io/docs/specs/semconv/http/http-metrics/), [convenções de métricas de banco](https://opentelemetry.io/docs/specs/semconv/db/database-metrics/) e [métricas Npgsql](https://www.npgsql.org/doc/diagnostics/metrics.html).

Fixar versões de instrumentação/exportadores e registrar mudanças de convenções semânticas. As convenções de pool ainda evoluem, e páginas do ASP.NET Core também descrevem comportamentos de .NET 11: usar o trecho aplicável ao .NET 10 e verificar a telemetria emitida, sem pressupor compatibilidade por nome.

**Pool Npgsql:** configurar nome lógico estável, como `expenses`, antes de exportar. A documentação informa que o nome padrão pode ser a connection string. Revisar `db.client.connection.pool.name` e outros atributos para impedir exposição de configuração, mesmo quando a senha já esteja omitida pelo driver.

Coletar também CPU, memória, GC/ThreadPool, reinícios, tasks, conexões e capacidade Aurora pelos instrumentos de runtime/infraestrutura disponíveis. Não criar uma implementação manual de cada contador nativo dentro da aplicação.

### Catálogo de negócio proposto

| Instrumento | Tipo / unidade | Quando registrar / dimensões permitidas |
| --- | --- | --- |
| `expenses.identity.users.created` | Counter / `{user}` | Uma criação confirmada; sem ID/e-mail. |
| `expenses.identity.logins` | Counter / `{attempt}` | Resultado final da tentativa; `outcome=success/rejected/blocked/failure`. |
| `expenses.identity.session.refreshes` | Counter / `{operation}` | Renovação; `outcome=success/rejected/failure`. |
| `expenses.identity.logouts` | Counter / `{operation}` | Logout; `outcome=success/failure`. |
| `expenses.identity.reconciliations.required` | Counter / `{reconciliation}` | Novo registro confirmado de divergência; `reason` de enum limitado. |
| `expenses.finance.closings` | Counter / `{closing}` | Novo fechamento confirmado, sem repetir em replay idempotente. |
| `expenses.finance.conflicts` | Counter / `{conflict}` | Rejeição por versão, concessão ou estado; `reason=version/lock/state`. |
| `expenses.idempotency.requests` | Counter / `{request}` | Decisão idempotente; `module`, `operation`, `result=new/replay/conflict/in_progress`. |
| `expenses.database.migrations` | Counter / `{execution}` | Resultado final do processo de migração; `result=success/failure`. |

Contadores em memória não são atômicos com a transação do PostgreSQL e podem perder medições em um crash. Servem para operação; relatórios financeiros e auditoria usam os registros persistidos. Não incluir `householdId`, `userId`, e-mail/hash, IP, trace/correlation/request ID, chave idempotente, timestamp, texto livre, SQL ou caminho concreto como dimensão de métrica.

Dimensões HTTP: método, status e **template de rota**; rota não identificada não deve ser substituída pela URL arbitrária. Recursos de ambiente/serviço são separados de labels por usuário. Definir allowlist de operações/causas e estimar combinações × buckets × réplicas antes de acrescentar uma dimensão. IDs podem aparecer como exemplars controlados quando suportados; não como labels.

### Histogramas e agregação

Buckets iniciais propostos em segundos: `0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.15, 0.25, 0.5, 0.75, 1, 2.5, 5, 10`, mais o bucket de overflow do backend. A inclusão de `0.15` ajuda a investigar a meta de 150 ms. Operações longas podem exigir outra configuração documentada.

Calcular p50/p95/p99 a partir do histograma agregado das réplicas e da janela escolhida. Não tirar média dos percentis de cada instância. Usar o count do histograma HTTP para volume/erros; métricas não devem seguir a amostragem dos traces. Ausência de tráfego é “sem dados”, não 100% de sucesso comprovado.

## 5. Traces e amostragem

Instrumentar o span servidor HTTP, dependências PostgreSQL/HTTP/Cognito e spans de casos de uso relevantes. Nomear operações de forma estável, como `finance.period.close`; não incluir UUID na identidade do span. Evitar um span por método ou dois spans para a mesma chamada SQL quando EF Core e Npgsql forem instrumentados juntos. `HttpClient` e SDKs devem propagar contexto pelos mecanismos suportados; não injetar headers próprios arbitrários no Cognito.

4xx esperados não devem ser marcados indiscriminadamente como falha do servidor no tracing; respeitar convenções da instrumentação. Registrar código de rejeição de domínio de forma limitada. Falhas 5xx/timeout precisam estar visíveis, mesmo quando tratadas no pipeline.

No .NET 10, `IExceptionHandler` que trata a exceção pode suprimir diagnósticos automáticos por padrão. Como o handler atual retorna `true`, testar a presença de log sanitizado, resultado HTTP e status do span, sem habilitar indiscriminadamente mensagens sensíveis ou duplicadas com `SuppressDiagnosticsCallback`. Referência: [tratamento de erros do ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling?view=aspnetcore-10.0).

**Amostragem a adotar:** em desenvolvimento/teste, coletar 100% dos traces durante validação. Para cumprir a intenção do FDD de preservar erros e reduzir sucessos, propor tail sampling no collector: reter erros e requests acima do limite da operação; taxa inicial de 10% para demais sucessos, a calibrar por volume/custo. Isso exige que spans cheguem ao collector, buffers dimensionados e ausência de descarte anterior que inviabilize a decisão.

Head sampling sozinho não garante “100% dos erros”, pois decide antes do desfecho. Se a implantação só oferecer esse modo, registrar explicitamente a limitação e manter métricas e logs de falha. A meta de retenção não garante entrega diante de falhas no pipeline. Referência: [OpenTelemetry Sampling](https://opentelemetry.io/docs/concepts/sampling/).

## 6. Dashboards, SLIs e alertas

| Dashboard | Visões mínimas |
| --- | --- |
| API | RPS, requests ativos, status 4xx/5xx, p50/p95/p99, abortos, operação/rota e versão implantada. |
| Dependências | Latência/erros Cognito e PostgreSQL, uso/máximo do pool, tempo de operações e timeouts. |
| Identity/Finance | Falhas/bloqueios de login, reconciliações, conflitos de versão e fechamentos confirmados. |
| Infraestrutura | Gateway/ALB, tasks, CPU/memória, reinícios, Aurora, backup/restauração e saúde do collector. |

Os alvos existentes no HLD são **99,9% de disponibilidade mensal** e **p95 < 150 ms para operações frequentes**, sujeitos a validação de carga/infraestrutura. São metas, não resultados medidos.

Definição inicial de SLIs para ratificar na implantação:

- **Disponibilidade:** `1 - falhas atribuíveis ao serviço / requests elegíveis`. No edge, considerar 5xx e timeouts de integração; incluir rejeição por saturação atribuível ao serviço. Excluir probes/Swagger e rejeições válidas de cliente, como entrada inválida ou falta de permissão. Classificar 429 contratual versus saturação para não esconder indisponibilidade.
- **Latência:** histograma por allowlist de operações frequentes; observar sucesso e falha separadamente e manter visão global. Importação/exportação não entram silenciosamente no mesmo alvo.
- **Ausência de tráfego:** usar probe sintético para confirmar alcance e detectar falha total; não substituir esse diagnóstico por divisão 0/0.

Usar uma camada de medição por SLI: Gateway para visão externa e backend para diagnóstico interno. Não somar contagens do mesmo request nas duas camadas. O histograma do backend não mede rede/navegador nem falhas antes de a API receber a chamada.

Alertas iniciais **propostos**, sem automação criada:

| Condição | Tratamento |
| --- | --- |
| Consumo acelerado do orçamento de erro | Alerta por múltiplas janelas; para SLO 99,9%, burn rate > 14,4 simultaneamente em 5 min e 1 h é um ponto de partida para severidade alta. Calibrar volume mínimo e operação de baixo tráfego. |
| p95 de operação frequente acima de 150 ms por 15 min | Aviso com volume suficiente; correlacionar dependência, versão e saturação. |
| Pool perto do máximo junto de espera/timeout | Alerta de capacidade; não alertar apenas porque conexões ociosas existem. |
| Nova reconciliação de identidade pendente | Encaminhar à operação responsável, com procedimento definido. |
| Falha de probes, perda de exportação ou ausência inesperada de telemetria | Alerta separado; não interpretar como sistema saudável. |

Burn rate compara a taxa observada de falhas ao orçamento permitido (`1 - SLO`); com 99,9%, 14,4 corresponde a 1,44% de falhas elegíveis. Alertas devem ter responsável, severidade, janela, critério de recuperação e runbook. Referência: [Google SRE — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/).

### Consulta de suporte e roteiro de investigação

Quando o schema JSON estiver chegando ao CloudWatch, esta consulta Logs Insights localiza os eventos do identificador informado pelo cliente:

```text
fields @timestamp, eventName, operation, statusCode, durationMs, errorCode, traceId, requestId
| filter correlationId = "97dfba56-bdf4-49ba-8f64-3cf9e14c134a"
| sort @timestamp asc
| limit 100
```

Selecionar os log groups e uma janela curta do incidente. A consulta pressupõe campos JSON extraídos no nível esperado; não é evidência de um coletor já instalado. Referência: [CloudWatch Logs Insights QL](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html).

Roteiro mínimo: localizar o request → verificar `errorCode`, versão e resultado → abrir o trace por `traceId` quando retido → comparar latência/erros da dependência e uso do pool no mesmo período → registrar diagnóstico e ação. Se o trace foi descartado pela política, usar os logs e métricas sem interpretar sua ausência como inexistência da falha.

## 7. Frontend, Liquibase e pipeline

**Frontend:** apresentar mensagem útil e `correlationId` em erro suportado; enviar esse identificador ao suporte. Não mostrar stack trace nem capturar formulário financeiro/senha na telemetria. Caso se adote telemetria de navegador, usar `service.name=expenses-ui`, rota normalizada e política de dados; erros JS, falhas de API e duração percebida são sinais distintos. A coleta ainda não existe.

**Liquibase:** cada execução controlada recebe `executionId`, versão do artefato, ambiente, resultado, duração e changeset/arquivo quando necessário. Nomes de changesets podem estar nos logs de implantação, não devem virar uma série de métrica por execução. Não registrar senha/URL JDBC completa nem dump de SQL com dados. O wrapper/pipeline pode normalizar saída; não presumir que Liquibase OSS já emite o schema JSON acima. A métrica de processo curto precisa ser entregue pelo pipeline/collector, não depender de um scrape após o processo terminar.

**Coleta:** evitar exportação síncrona por request; usar mecanismos suportados de buffering/exportação com limites e observar descartes. Falha do coletor não deve derrubar um GET de negócio. Flush no encerramento deve respeitar prazo limitado. Esse envio técnico de telemetria não muda o modelo síncrono dos casos de uso.

## 8. Relação com o FDD e adoção

Os nomes antigos de métricas do FDD descrevem a intenção. Para implementação nova, consolidar a nomenclatura:

| FDD | Instrumento/visão alvo |
| --- | --- |
| `identity_user_created_total` | `expenses.identity.users.created`. |
| `identity_login_success/failure/blocked_total` | `expenses.identity.logins` com outcome. |
| `identity_session_refresh_total`, `identity_logout_total` | `expenses.identity.session.refreshes`, `expenses.identity.logouts`. |
| `identity_endpoint_latency_ms`, `identity_errors_total` | Histograma HTTP em segundos, percentis e contagens por status. |
| `identity_cognito_operation_errors_total` | `expenses.dependency.duration` filtrado por Cognito/outcome. |
| `identity_profile_reconciliation_needed_total` | `expenses.identity.reconciliations.required`. |

Não emitir as duas famílias simultaneamente e somá-las. Se algum consumidor futuro exigir alias, documentar a tradução no exporter/dashboard. Atualizar o FDD na adoção, incluindo pseudonimização e a implementação real de sampling.

Sequência recomendada: padronizar correlação e erros → definir logger JSON e sanitização → habilitar instrumentos/exportação em teste → conferir cardinalidade e duplicação → criar dashboards/alertas → validar carga, retenção e procedimentos operacionais.

## 9. Critérios de aceite

- [ ] Um erro de API é encontrado por `correlationId` e ligado ao trace por `traceId` correto.
- [ ] Log JSON tem schema/tipos estáveis e apenas um resumo terminal por request.
- [ ] Dados sintéticos sensíveis de teste não aparecem em logs, spans, atributos de pool ou métricas.
- [ ] `400`, `409`, `500`, `503`, timeout e cancelamento recebem classificação coerente.
- [ ] Erro tratado no .NET 10 continua observável, sem duplicação de exceções.
- [ ] Histogramas usam segundos, têm bucket de 150 ms e agregam corretamente réplicas.
- [ ] Replay não incrementa novamente criação/fechamento; contadores não são usados como auditoria financeira.
- [ ] IDs e URLs arbitrários não aumentam a cardinalidade das métricas.
- [ ] Falha do collector não impede negócio; perda de telemetria é detectável.
- [ ] Sampling, SLOs, responsáveis, retenção e alertas foram validados no ambiente alvo.
