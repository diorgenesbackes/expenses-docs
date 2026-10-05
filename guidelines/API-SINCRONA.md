# Padrão de contratos para APIs síncronas

[Índice](README.md) · [Backend](BACKEND.md) · [Frontend](FRONTEND.md) · [Observabilidade](OBSERVABILIDADE.md)

**Status:** padrão para novos contratos e evolução planejada dos existentes. Este documento não altera endpoints nem configurações em execução. Exceções já especificadas no FDD e diferenças do código atual estão registradas na seção 9.

## 1. Decisões de formato

| Tema | Padrão do Expenses |
| --- | --- |
| Transporte | HTTP sobre TLS em ambientes publicados; JSON UTF-8. |
| Versionamento | Negócio sob `/v1`; health fora desse prefixo. |
| Request | Objeto com os campos da operação, sem envelope `request` ou `data`. |
| Sucesso com um recurso | Objeto com os campos públicos do recurso. |
| Sucesso composto | Objeto com propriedades de negócio nomeadas, como `user` e `session`, quando exigido pelo caso de uso. |
| Coleção paginada | `{ items, page, pageSize, totalCount }`. |
| Sucesso sem representação | `204`, sem corpo. |
| Erro | `application/problem+json`, com Problem Details e extensões padronizadas. |
| Correlação | `X-Correlation-Id` na resposta; identificadores de diagnóstico no erro e nos logs. |

Não criar um envelope universal com `success`, `statusCode`, `message` e `data`: o status pertence ao HTTP, e o corpo descreve o resultado. Essa é uma decisão do projeto que preserva a paginação existente. Um serviço compartilhado de serialização não deve envolver automaticamente toda resposta, especialmente arquivos, health e `204`.

“Síncrona” significa que a resposta informa o resultado da operação executada durante a chamada. Não responder `202` prometendo um job inexistente. `async/await` continua apropriado para I/O. Uma falha de transporte pode ocorrer depois do commit: ausência de resposta não prova que a operação deixou de acontecer.

## 2. Requests

### Método, rota e entrada

| Método | Uso e resultado esperado |
| --- | --- |
| GET | Consulta; filtros/paginação na query, sem corpo e sem mutação de negócio. |
| POST | Criação ou comando explícito; `201` para recurso criado, `200` para resultado de comando ou `204` quando não houver representação. |
| PUT | Substituição dos campos editáveis documentados; declarar a semântica de campos ausentes. Não usar para atualização parcial implícita. |
| PATCH | Alteração parcial somente com formato, campos permitidos e semântica de `null` definidos no contrato. Não presumir JSON Patch ou Merge Patch sem escolhê-los. |
| DELETE | Exclusão conforme ciclo de vida; normalmente `204`. Documentar comportamento quando o recurso já não existir. |

Usar substantivos no plural e IDs opacos nas rotas; comandos de domínio podem ser recursos, como fechamentos de uma competência. Cada endpoint deve documentar seu `operationId` estável no OpenAPI. A semântica HTTP segue a [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html); nomes e formatos de corpo são escolhas deste projeto.

Path identifica o recurso; query ajusta a consulta; body contém os campos editáveis. Não repetir a mesma identidade em path e body. Não aceitar do cliente ator autenticado, papéis, timestamps de auditoria ou totais calculados pelo servidor. A casa indicada na rota deve ser autorizada no backend.

### Convenções de dados

- Propriedades JSON e query parameters em `camelCase`; enums textuais em `snake_case`, com valores documentados. Não serializar enums como números implícitos.
- IDs atuais como strings UUID; consumidores tratam IDs como opacos. Não mudar silenciosamente para prefixos como `usr_`.
- Instantes como ISO 8601 em UTC; nas novas respostas, emitir `Z`, por exemplo `2026-10-05T14:30:00Z`. Aceitar offsets válidos e normalizar quando o campo representar um instante. Preservar o contrato de data civil/competência, sem conversão de fuso.
- Booleanos são `true`/`false`; coleções vazias são `[]`, nunca `null`.
- Declarar obrigatoriedade, nulabilidade, comprimento, escala, faixa e normalização de cada campo. Campo obrigatório ausente ou nulo é inválido; em PATCH, ausente e nulo precisam ter significados distintos e explícitos.
- Novos comandos devem rejeitar propriedades desconhecidas para evitar erro de digitação e mass assignment; consumidores de respostas devem tolerar propriedades adicionais. Isso requer configuração e teste na implementação, não apenas DTOs tipados.
- Senhas/tokens não passam por trim ou transformação genérica. Normalização de e-mail segue a regra do domínio e do provedor.

**Formato financeiro recomendado para a primeira implementação:** decimal canônico textual, como `"1234.56"`, separado de `currencyCode: "BRL"`; duas casas para campos monetários de escala 2, sem separador de milhar ou vírgula. Usar sinal apenas onde o modelo permite negativos. Versões `bigint` de edição devem ser strings decimais positivas, com limite de `Int64`, como `"7"`. Essas escolhas ainda precisam ser formalizadas com o primeiro contrato financeiro; não convertem automaticamente o `totalCount` numérico existente. A política de arredondamento permanece uma decisão de negócio.

### Headers

| Header | Regra |
| --- | --- |
| `Content-Type` | `application/json` para body JSON; não enviar em GET vazio. Formatos de PATCH/arquivo devem ser documentados separadamente. |
| `Accept` | Cliente aceita `application/json` e `application/problem+json`; respostas de arquivo declaram seu media type. |
| `X-Correlation-Id` | Opcional no request, UUID canônico; a API devolve o valor válido ou gera outro. Ver seção 7. |
| `traceparent` / `tracestate` | Contexto técnico de tracing, validado/propagado pela instrumentação; não é credencial nem chave de negócio. |
| `Idempotency-Key` | Aplicável somente aos comandos que implementem esse contrato; UUID aleatório, sem dados pessoais. |
| Cookie / proteção CSRF | Seguir a sessão definida com Identity. Não inventar bearer token no navegador em paralelo ao FDD. |

Se frontend e API tiverem origens diferentes, a política CORS deve permitir os headers necessários e expor `X-Correlation-Id`/`Location` para leitura no navegador quando usados. Definir origens explícitas; isso não substitui CSRF nem autenticação.

Cada endpoint DEVE declarar limite de body e orçamento de execução. Identity já especifica **8 KB**, orçamento total de **5 s** e chamada Cognito de **3 s**; importações/exportações precisam de limites próprios. Rejeitar payload excedente com `413`; não usar limites ilimitados ou o tamanho de um arquivo para justificar um job não previsto no HLD.

## 3. Respostas de sucesso

GET paginado atual, disponível apenas em Development:

```http
GET /v1/identity/users?page=1&pageSize=20 HTTP/1.1
Accept: application/json, application/problem+json
```

Exemplo de corpo compatível com a listagem atual, com dados sintéticos:

```json
{
  "items": [
    {
      "id": "00000000-0000-0000-0000-000000000001",
      "email": "usuario@example.com",
      "createdAt": "2026-10-05T14:30:00Z",
      "updatedAt": "2026-10-05T14:30:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1
}
```

Paginação padrão: `page >= 1`, `pageSize` de 1 a 100, defaults 1 e 20; verificar overflow do offset. `totalCount` é a contagem filtrada e autorizada, antes de paginar. No diagnóstico atual, o escopo é global por sua finalidade local. Coleção vazia ou página além do fim retorna `200` com `items: []`, preservando o total.

Ordenação deve ser estável com desempate único. A listagem atual ordena somente por ID. Se uma nova rota aceitar ordenação, adotar `sort=-createdAt,id`, com lista permitida de campos, sem interpolação direta em SQL. Não adicionar esse parâmetro à documentação do endpoint atual como se já existisse. Contagem e itens sem snapshot podem divergir durante escrita concorrente.

Para coleção pequena e finita sem paginação, usar `{ "items": [...] }` e documentar o limite. Não retornar milhões de linhas usando essa exceção.

### Exemplo de criação

Exemplo ilustrativo de contrato futuro; o endpoint de casas ainda não está implementado. Demonstra body de entrada, campos gerados e headers, sem substituir a especificação funcional do cadastro.

```http
POST /v1/households HTTP/1.1
Content-Type: application/json
Accept: application/json, application/problem+json
X-Correlation-Id: 97dfba56-bdf4-49ba-8f64-3cf9e14c134a
Idempotency-Key: b5d1dd15-82e9-4cb0-8d42-dad95cb9ab0b
```

Request:

```json
{
  "name": "Casa principal",
  "currencyCode": "BRL",
  "timeZone": "America/Sao_Paulo"
}
```

Response:

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/households/00000000-0000-0000-0000-000000000010
Cache-Control: no-store
X-Correlation-Id: 97dfba56-bdf4-49ba-8f64-3cf9e14c134a
```

```json
{
  "id": "00000000-0000-0000-0000-000000000010",
  "name": "Casa principal",
  "currencyCode": "BRL",
  "timeZone": "America/Sao_Paulo",
  "createdAt": "2026-10-05T14:30:00Z"
}
```

A sessão e a proteção CSRF exigidas pelo endpoint são pressupostos do exemplo, omitidos para não expor credenciais. O ator criador vem da sessão; não é um campo livre do request. A chave idempotente só deve ser anunciada quando sua semântica estiver implementada.

Criação retorna `201` com representação autorizada e `Location` quando houver URI de recurso consultável. Comando que produz um resultado sem criar recurso endereçável pode retornar `200`. `204` não tem JSON, nem mesmo `{}`. Definir `Cache-Control: no-store` para respostas autenticadas/sensíveis e erros, salvo política de cache específica revisada. Cookies e tokens nunca aparecem no objeto de sucesso como conveniência para JavaScript.

## 4. Erros e catálogo inicial

Adotar Problem Details conforme a [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html). O status HTTP é a fonte de verdade e deve coincidir com `status` no corpo. Não devolver erro de negócio dentro de `200`. Extensões são decisões do Expenses; não são campos obrigatórios impostos pela RFC.

| Campo | Regra do projeto |
| --- | --- |
| `type` | Inicialmente `about:blank`; tipos específicos só com URI estável documentada. |
| `title` | Título estável do status, como `Bad Request` ou `Conflict`, quando `type` é `about:blank`. |
| `status` | Mesmo inteiro do status HTTP. |
| `detail` | Explicação segura para o usuário; opcional em erros internos. Não é código de decisão do cliente. |
| `code` | Código estável em `SCREAMING_SNAKE_CASE`, definido no catálogo do endpoint. |
| `traceId` | ID W3C da Activity do request; 32 caracteres hexadecimais. Adoção pendente no código atual. |
| `correlationId` | Mesmo UUID devolvido em `X-Correlation-Id`. |
| `errors` | Somente validação: objeto de caminho de campo → array de mensagens; `$` para erro geral/body inválido. Caminhos em camelCase, como `items[0].amount`. |
| `instance` | Opcional; identificador opaco da ocorrência, sem incluir URL/query com dados pessoais. |

Exemplo **alvo**, ainda não emitido integralmente pelo código atual:

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "Revise os campos indicados.",
  "code": "REQUEST_VALIDATION_FAILED",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "correlationId": "97dfba56-bdf4-49ba-8f64-3cf9e14c134a",
  "errors": {
    "pageSize": ["Informe um valor entre 1 e 100."]
  }
}
```

| HTTP | Código inicial | Quando usar |
| --- | --- | --- |
| `400` | `REQUEST_VALIDATION_FAILED` | JSON, parâmetros, campos ou limites inválidos. |
| `401` | `IDENTITY_AUTHENTICATION_REQUIRED` | Sessão ausente/expirada; credenciais inválidas podem usar `IDENTITY_INVALID_CREDENTIALS`. Preservar challenge do esquema HTTP quando aplicável. |
| `403` | `RESOURCE_ACCESS_DENIED` | Ator autenticado sem permissão. |
| `404` | `RESOURCE_NOT_FOUND` | Recurso ausente ou oculto pela política documentada. |
| `409` | `FINANCE_PERIOD_VERSION_CONFLICT` | Versão desatualizada; outros conflitos têm códigos específicos. |
| `409` | `IDEMPOTENCY_KEY_REUSED` | Mesma chave com conteúdo diferente. |
| `413` | `REQUEST_TOO_LARGE` | Tamanho acima do limite. |
| `415` | `UNSUPPORTED_MEDIA_TYPE` | Formato de body não suportado. |
| `422` | `IDENTITY_PASSWORD_POLICY_VIOLATION` | Exceção de domínio já prevista pelo FDD; não alternar arbitrariamente entre 400/422. |
| `423` | `IDENTITY_LOGIN_BLOCKED` | Bloqueio temporário já previsto pelo FDD. |
| `429` | `REQUEST_RATE_LIMITED` | Limitação de frequência. |
| `500` | `UNEXPECTED_ERROR` | Falha interna, inclusive schema/configuração inválidos quando não forem indisponibilidade transitória. |
| `503` | `DEPENDENCY_UNAVAILABLE` | Dependência indisponível ou timeout interno reconhecido. |

Novos códigos devem indicar módulo/causa estável, sem valores dinâmicos. Documentar também erros produzidos pelo pipeline, como `405`/`406`; produzir Problem Details quando a resposta ainda puder ser escrita. Falhas rejeitadas no Gateway/proxy podem ter outro formato: configurar o edge quando possível e manter fallback de erro no cliente.

Enviar `Retry-After` para `429`, `423` ou `503` apenas quando houver prazo conhecido; seu significado é espera mínima, não garantia de que repetir uma escrita seja seguro. `504` costuma vir do gateway aguardando upstream; a API não deve transformar todo timeout em `504`.

## 5. Concorrência e repetição de comandos

No contrato de escritas mensais, documentar a versão esperada e a concessão de edição. Proposta para a primeira implementação: campos `expectedVersion` e `editLockToken`; o token de concessão não vai para logs. A comparação e o incremento ocorrem na transação do caso de uso. Um `409` pode informar a necessidade de recarregar, sem expor dados de uma casa não autorizada.

Para cada comando com `Idempotency-Key`, definir obrigatoriedade, escopo, prazo de retenção e fingerprint canônico. O padrão para novas implementações é:

1. Mesmo ator/casa/operação/chave e conteúdo já concluído: devolver o resultado persistido e autorizado, sem nova gravação de negócio.
2. Mesma chave e conteúdo diferente: `409 IDEMPOTENCY_KEY_REUSED`.
3. Mesma operação ainda em andamento: `409 IDEMPOTENCY_REQUEST_IN_PROGRESS`; a consulta/repetição segue o contrato, sem iniciar uma segunda execução.
4. Resultado desconhecido por perda de resposta: repetir com a mesma chave dentro do prazo documentado.

Autenticar e autorizar também o replay. A chave não é credencial. Não armazenar senha, cookie ou token para reconstruir uma resposta. Sessões/cadastro exigem desenho específico de repetição e compensação; o header isolado não cria a garantia. Se a chave não for suportada, isso deve estar claro no OpenAPI.

## 6. Registro mínimo de um endpoint

Antes de implementar, registrar no FDD/OpenAPI:

- Método, rota, `operationId`, módulo e finalidade.
- Autenticação, autorização por recurso e proteção CSRF aplicáveis.
- Schema de path/query/body, campos aceitos e normalização.
- Headers de entrada/saída, tipos de conteúdo e política de cache.
- Respostas de sucesso, erros/códigos e exemplos com dados sintéticos.
- Paginação/ordenação, concorrência e idempotência quando aplicáveis.
- Limite de payload e orçamento de tempo, incluindo dependências.
- `operation` de observabilidade, métricas utilizadas e classificação de dados sensíveis.

Alterações incompatíveis exigem nova versão ou migração coordenada antes de publicar. Adicionar campo opcional à resposta pode ser compatível; remover/renomear campo, trocar número por string, tornar request obrigatório ou mudar semântica de enum exige análise explícita.

## 7. Correlação entre resposta e telemetria

Padrão proposto: aceitar um único `X-Correlation-Id` em formato UUID canônico; ausente, inválido ou múltiplo é substituído por UUID gerado no servidor. Nunca refletir texto arbitrário ou guardar o valor rejeitado. Devolver o identificador em respostas de sucesso/erro alcançadas pela API, inclusive validação e 404.

`correlationId` serve à busca de suporte e pode abranger tentativas relacionadas. `traceId` identifica o trace técnico; `requestId` distingue a requisição no servidor. Não usá-los como chave de idempotência, autorização ou label de métrica. Os detalhes de W3C, propagação, confiança e logs ficam no [padrão de observabilidade](OBSERVABILIDADE.md).

## 8. Critérios de aceite

- [ ] OpenAPI corresponde ao JSON, aos headers e aos limites reais.
- [ ] Sucesso, coleção vazia, `204`, validação e falhas relevantes foram testados.
- [ ] Erros de MVC, middleware e negócio usam o mesmo contrato alvo.
- [ ] `status`, `code`, correlação e trace permitem encontrar a ocorrência no log.
- [ ] Isolamento entre casas, conflito e replay foram testados quando aplicáveis.
- [ ] Nenhum segredo, SQL ou stack trace aparece na resposta.
- [ ] Cliente tolera erro sem JSON vindo do edge e não repete escrita cegamente.

## 9. Compatibilidade e adoção pendente

| Situação | Tratamento |
| --- | --- |
| Listagem atual | Manter `{ items, page, pageSize, totalCount }`, UUIDs e disponibilidade só em Development. |
| Health atual | Manter `{ "status": "up" }` ou `{ "status": "down" }`; `/health/db` usa `200/503`. Não envolver nem converter para Problem Details. |
| Erros atuais | Já usam Problem Details, mas faltam `code` e correlação padronizada; `type` e erros MVC ainda precisam ser alinhados. |
| `traceId` atual | O handler usa `HttpContext.TraceIdentifier`; migrar para `Activity.TraceId` com teste e identificação separada de `requestId`. |
| Identity no FDD | Preservar respostas compostas `user`/`session`, `422` e `423`. O FDD ainda tem exemplos de `{}` para GET/204 e IDs prefixados: corrigir esses pontos na revisão do contrato antes de implementá-lo. |
| Dinheiro, versão e idempotência | Padronização recomendada acima; formalizar junto dos endpoints de negócio, ainda não existentes. |

O guia especifica o padrão e os passos de compatibilização. Não anuncia que headers, middleware ou serializadores já foram alterados.
