### FDD: Criação de usuário e autenticação

[1 — Planejamento](1-planejamento.md) · [2 — Desenvolvimento](2-desenvolvimento.md) · [3 — Testes](3-testes.md) · [4 — Operação](4-operacao.md)

Versão: 1.2
Data: 2026-10-07
Responsável: Diórgenes Backes

Estado: cadastro E2 e login/sessões E3/E4 implementados no backend; interface e homologação ainda pendentes.
Contrato operacional de sessões, CSRF, cookies e concorrência: [E3/E4](2-desenvolvimento.md#login-e-sessoes-e3-e4).
Decisões aprovadas: UUID, bloqueio exclusivamente por e-mail, encerramento em qualquer falha de renovação, revogação imediata na API e p95 < 150 ms em todas as operações. A [entrega E2](2-desenvolvimento.md#cadastro-e2) registra evidências e limites; o desempenho local medido não atende à meta.

---

### 1. Contexto e motivação técnica
A feature de criação de usuário e autenticação cria a base segura de acesso ao
produto Contas da Casa. Ela substitui o acesso informal ao controle financeiro
por um fluxo autenticado, com sessão protegida, criação de perfil interno e
integração controlada com o Amazon Cognito.

No HLD, o frontend React consome APIs REST JSON versionadas em `/v1`. Para esta
feature, o frontend não chama o Cognito diretamente. Todas as operações de
cadastro, login, logout e validação de sessão passam pelo `Identity Service`,
que é responsável por intermediar chamadas ao Cognito, criar o perfil interno do
usuário e escrever cookies seguros.

Atores principais:
- Usuário final que cria conta e realiza login.
- Frontend web React.
- `Identity Service`.
- Amazon Cognito User Pools.
- Aurora PostgreSQL Serverless v2, schema `identity`.

Limites da feature:
- A feature autentica usuários e valida sessão.
- Papéis por casa não ficam no token.
- Papéis por casa serão carregados posteriormente pelo domínio de Household.
- Recuperação de senha fica fora do MVP.
- Confirmação de e-mail fica fora do MVP.
- O sistema pode lembrar o e-mail no dispositivo, mas nunca armazena senha no
  frontend.

Suposições e restrições:
- O usuário informa apenas e-mail e senha no cadastro.
- Senha mínima: 8 caracteres, pelo menos uma letra e um número.
- Access token expira em `15 minutos`.
- Refresh token expira em `7 dias`.
- Tokens são mantidos apenas em cookies `HttpOnly`, `Secure` e `SameSite`.
- O Cognito é acessado somente pelo `Identity Service`.
- A meta aprovada é `p95 < 150 ms` em todas as operações, incluindo dependências externas; carga, ambiente e ponto de medição devem acompanhar a evidência.

---

### 2. Objetivos técnicos
- Criar usuário autenticável no Cognito e perfil interno no schema `identity`,
  mantendo vínculo consistente por `cognitoSub`.
- Autenticar usuários sem expor tokens ao JavaScript do frontend.
- Manter sessão com access token de `15 minutos` e refresh token de `7 dias` em
  cookies seguros.
- Invalidar imediatamente a sessão na API por registro persistente; tentar também revogar tokens no Cognito. Falha na persistência local impede confirmar o logout.
- Validar sessão atual e renovar access token quando o refresh token ainda for
  válido.
- Limitar brute force com bloqueio temporário após `10` tentativas inválidas de
  login em `15 minutos`.
- Avisar o usuário sobre o limite somente nas `3` últimas tentativas.
- Não registrar senha, tokens ou valores sensíveis em logs, métricas ou traces.

---

### 3. Escopo e exclusões

**Incluído**
- Cadastro de usuário com e-mail e senha.
- Criação de conta autenticável no Amazon Cognito.
- Criação do perfil interno no `Identity Service`.
- Login com e-mail e senha.
- Logout com limpeza de cookies e revogação de tokens no Cognito.
- Validação de sessão atual.
- Renovação de sessão usando refresh token seguro.
- Lembrar e-mail no dispositivo ou navegador, quando o usuário optar por isso.
- Proteção contra brute force com limite de tentativas.
- Observabilidade de eventos de autenticação.

**Excluído**
- Confirmação de e-mail no cadastro.
- Recuperação de senha.
- Login social.
- MFA.
- Armazenamento de senha no navegador.
- Retorno de papéis por casa no token ou na sessão de identidade.
- Gerenciamento de papéis por casa.
- Convite por e-mail.
- Cadastro de casa e vínculo de usuário à casa.

---

### 4. Fluxos detalhados e diagramas
**Fluxo principal**
- O usuário acessa a tela de cadastro.
- O usuário informa e-mail e senha.
- O frontend envia `POST /v1/identity/users`.
- O API Gateway encaminha a requisição ao `Identity Service`.
- O `Identity Service` valida e-mail, política de senha e idempotência da
  operação.
- O `Identity Service` cria o usuário no Cognito sem exigir confirmação de
  e-mail no MVP.
- O `Identity Service` persiste o perfil interno no schema `identity`, vinculado
  ao `cognitoSub`.
- O sistema retorna `201 Created`.
- O usuário realiza login informando e-mail e senha.
- O frontend envia `POST /v1/identity/sessions`.
- O `Identity Service` verifica rate limit e autentica no Cognito.
- O `Identity Service` grava cookies seguros de sessão.
- O sistema retorna o perfil interno básico.
- O frontend passa a acessar rotas autenticadas usando cookies.
- Ao validar sessão, o frontend chama `GET /v1/identity/session/current`.
- Se o access token estiver válido, o `Identity Service` retorna a sessão atual.
- Se o access token estiver expirado e o refresh token válido, o
  `Identity Service` renova a sessão e atualiza os cookies.
- Ao realizar logout, o frontend chama `DELETE /v1/identity/sessions/current`.
- O `Identity Service` confirma revogação local persistente, tenta revogar tokens no Cognito e limpa cookies.

**Fluxos alternativos e exceções**
- Se o usuário optar por lembrar e-mail, o frontend armazena somente o e-mail
  no dispositivo ou navegador.
- Se o cadastro no Cognito tiver sucesso e a persistência do perfil falhar, o
  `Identity Service` só tenta remover o principal comprovado antes de commit ambíguo; caso contrário preserva a conta para reconciliação.
- Se a remoção compensatória falhar, o sistema registra evento de reconciliação
  sem expor dados sensíveis.
- Se o login for inválido, o contador de tentativas apenas por `emailHash` é
  incrementado.
- Nas `3` últimas tentativas antes do bloqueio, a resposta informa tentativas
  restantes.
- Após `10` tentativas inválidas em `15 minutos`, o login é bloqueado por `15
  minutos`.
- Se a sessão não puder ser renovada, o sistema encerra a sessão e limpa cookies: `401` para credencial inválida, `503` para indisponibilidade.
- Se o logout não conseguir revogar tokens no Cognito, o sistema limpa cookies e
  registra falha operacional para investigação.

**Diagramas** (opcional)
- Sequência de cadastro: Frontend -> API Gateway -> Identity Service -> Cognito
  -> Aurora schema `identity` -> Frontend.
- Sequência de login: Frontend -> API Gateway -> Identity Service -> rate limit
  -> Cognito -> cookies seguros -> Frontend.
- Sequência de validação: Frontend -> Identity Service -> valida access token ou
  usa refresh token -> atualiza cookies -> Frontend.
- Estados da sessão: `anonymous` -> `authenticated` -> `refreshing` ->
  `authenticated` ou `expired`.

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)
**Criar usuário**
- Tipo: endpoint
- Assinatura/Rota: `POST /v1/identity/users`
- Método: `POST`
- Semântica de status/headers:
  - `201 Created`: usuário criado no Cognito e perfil interno persistido.
  - `400 Bad Request`: payload inválido.
  - `409 Conflict`: e-mail já cadastrado.
  - `422 Unprocessable Entity`: senha não atende à política mínima.
  - `500 Internal Server Error`: falha interna sanitizada.
  - `503 Service Unavailable`: dependência indisponível ou resultado incerto que exige reconciliação.
  - `413` para corpo acima de 8 KB; `415` para Content-Type incompatível; `403` para origem não permitida.
  - `X-Correlation-Id`: identificador de correlação da requisição.
  - `Idempotency-Key`: UUID opcional e recomendado. Mesma chave/e-mail recupera resultado por 24 horas da reserva; não altera senha. Mesma chave/outro e-mail, chave expirada ou operação em execução: `409`. Nova chave/e-mail existente: `409`. Operação incerta não é reexecutada nem descartada por expiração: `503`.
  - E-mail ASCII normalizado com até 128 caracteres; senha de 8 a 256 caracteres com letra e algarismo ASCII, sem espaços/controles. Senha não é normalizada nem persistida.
  - JSON rejeita propriedades desconhecidas. `Cache-Control: no-store` em sucesso/erro; erros ProblemDetails contêm `code`, `traceId` W3C e `correlationId` UUID. Cadastro não inicia sessão nem retorna Location.

**Exemplo de requisição**
```json
{
  "email": "usuario@exemplo.com",
  "password": "senha123"
}

```

**Exemplo de resposta**

```json
{
  "user": {
    "id": "1a44c92b-6af1-4e8d-913f-6372937bdf10",
    "email": "usuario@exemplo.com",
    "createdAt": "2026-06-03T12:00:00Z"
  }
}
```

**Criar sessão**
- Tipo: endpoint
- Assinatura/Rota: `POST /v1/identity/sessions`
- Método: `POST`
- Semântica de status/headers:
  - `200 OK`: login realizado e cookies seguros definidos.
  - `400 Bad Request`: payload inválido.
  - `401 Unauthorized`: credenciais inválidas.
  - `423 Locked`: login bloqueado temporariamente por excesso de tentativas.
  - `503 Service Unavailable`: Cognito indisponível ou timeout de autenticação.
  - `X-CSRF-Token`: obrigatório, obtido em `GET /v1/identity/csrf` junto do cookie CSRF.
  - `Set-Cookie`: grava cookies `cc_at` e `cc_rt` com `HttpOnly`, `Secure` e
    `SameSite`.
  - `X-Correlation-Id`: identificador de correlação da requisição.

**Exemplo de requisição**
```json
{
  "email": "usuario@exemplo.com",
  "password": "senha123"
}

```

**Exemplo de resposta**

```json
{
  "user": {
    "id": "1a44c92b-6af1-4e8d-913f-6372937bdf10",
    "email": "usuario@exemplo.com"
  },
  "session": {
    "authenticated": true,
    "accessTokenExpiresInSeconds": 900,
    "refreshTokenExpiresInSeconds": 604800
  }
}
```

**Encerrar sessão atual**
- Tipo: endpoint
- Assinatura/Rota: `DELETE /v1/identity/sessions/current`
- Método: `DELETE`
- Semântica de status/headers:
  - `204 No Content`: revogação local confirmada, cookies removidos e revogação no Cognito executada ou tentada. Novas validações rejeitam a sessão imediatamente; requisições já autorizadas terão semântica definida em E4.
  - `401 Unauthorized`: sessão ausente ou inválida.
  - `500 Internal Server Error`: falha interna; `503` se não for possível confirmar revogação local por indisponibilidade. Não confirmar logout sem revogação local.
  - `X-CSRF-Token`: obrigatório, junto do cookie CSRF.
  - `Set-Cookie`: expira cookies `cc_at` e `cc_rt`.
  - `X-Correlation-Id`: identificador de correlação da requisição.

**Exemplo de requisição**
```json
{}

```

**Resposta:** `204` sem corpo.

**Consultar sessão atual**
- Tipo: endpoint
- Assinatura/Rota: `GET /v1/identity/session/current`
- Método: `GET`
- Semântica de status/headers:
  - `200 OK`: sessão válida ou renovada.
  - `401 Unauthorized`: sessão ausente, expirada ou não renovável.
  - `503 Service Unavailable`: Cognito indisponível durante tentativa de
    renovação.
  - `X-CSRF-Token`: obrigatório também no GET que pode renovar sessão.
  - `409 Conflict`: outra renovação está em andamento; não limpar cookies.
  - `Set-Cookie`: pode atualizar `cc_at` quando houver renovação.
  - `X-Correlation-Id`: identificador de correlação da requisição.

**Exemplo de requisição**
```json
{}

```

**Exemplo de resposta**

```json
{
  "user": {
    "id": "1a44c92b-6af1-4e8d-913f-6372937bdf10",
    "email": "usuario@exemplo.com"
  },
  "session": {
    "authenticated": true,
    "refreshed": false,
    "accessTokenExpiresInSeconds": 620
  }
}
```

Limites aplicáveis aos contratos:
- Login: até `10` tentativas inválidas em `15 minutos` apenas por `emailHash`.
- Bloqueio de login: `15 minutos`.
- Tamanho máximo de payload: `8 KB`.
- Timeout interno para chamadas ao Cognito: `3 segundos`.
- Orçamento máximo de processamento por endpoint: `5 segundos`.
- Latência operacional esperada: `p95 < 150 ms` para todas as operações.
- Versionamento: rotas sob `/v1`; mudanças incompatíveis exigem nova versão.

---

### 6. Erros, exceções e fallback

- Matriz de erros previstos e tratamentos
  - Payload inválido: retornar `400`, registrar `failureReason=invalid_payload`
    e não chamar Cognito.
  - Senha fora da política: retornar `422`, registrar
    `failureReason=password_policy`.
  - E-mail já cadastrado: retornar `409`, registrar `failureReason=user_exists`.
  - Credenciais inválidas: retornar `401`, incrementar contador de tentativas e
    não revelar se o e-mail existe.
  - Limite de tentativas atingido: retornar `423` e informar tempo de bloqueio.
  - Access token expirado com refresh válido: renovar sessão e retornar `200`.
  - Access token expirado com refresh inválido: limpar cookies e retornar `401`.
  - Cognito indisponível: retornar `503`, registrar erro operacional e não
    executar fallback inseguro.
  - Perfil interno ausente após autenticação válida: retornar `503` e registrar reconciliação por `cognitoSub`; não vincular conta por e-mail nem criar perfil implicitamente.
  - Falha na persistência do perfil após criação no Cognito: compensar somente principal comprovadamente criado pela operação e antes de commit ambíguo; caso contrário registrar reconciliação e não excluir a conta.

- Estratégias de resiliência do cadastro: sem retries automáticos; três segundos por chamada Cognito e orçamento de cinco segundos do caso de uso após leitura HTTP. Recuperação por idempotência e reconciliação explícita; sem filas/jobs ou circuit breaker dedicado.
- Política de fallback
- Não autenticar usuário sem validação bem-sucedida pelo Cognito.
  - Não criar perfil interno sem usuário correspondente no Cognito.
  - Não retornar tokens no corpo da resposta.
  - Em falha de renovação, limpar cookies e exigir novo login.
  - Em divergência Cognito versus perfil interno, priorizar segurança,
    registrar reconciliação e bloquear acesso se a sincronização não for
    confiável.
- Invariantes: senha nunca é armazenada no frontend; tokens nunca são expostos
  ao JavaScript; sessão válida exige usuário Cognito e perfil interno; papéis
  por casa não são retornados pela autenticação; logs nunca incluem senha ou
  tokens.

---

### 7. Observabilidade

**Métricas**

- `identity_user_created_total`
- `identity_login_success_total`
- `identity_login_failure_total`
- `identity_login_blocked_total`
- `identity_session_refresh_total`
- `identity_logout_total`
- `identity_endpoint_latency_ms` por endpoint com `p50`, `p95` e `p99`
- `identity_errors_total` por endpoint e status
- `identity_cognito_operation_errors_total`
- `identity_profile_reconciliation_needed_total`

**Logs**

- Formato JSON estruturado.
- Campos essenciais: `correlationId`, `userId`, `emailHash`, `ipHash`,
  `userAgent`, `endpoint`, `statusCode`, `authEvent`, `failureReason` e
  `timestamp`.
- Campos proibidos: senha, access token, refresh token, ID token, código secreto
  ou payload sensível do Cognito.

**Tracing**

- Spans principais: `identity.request`, `auth.rate_limit.check`,
  `cognito.operation`, `profile.persistence`, `session.cookie.write`.
- Amostragem: `100%` para erros e amostragem reduzida para fluxos de sucesso.
- Cada trace deve propagar `correlationId` entre API Gateway, Identity Service e
  chamadas ao Cognito quando tecnicamente aplicável.

**Dashboards e alertas**

- Dashboard de cadastros e logins.
- Dashboard de falhas de login e bloqueios.
- Dashboard de latência por endpoint.
- Alerta para crescimento anormal de `401`, `423` ou `5xx`.
- Alerta para falhas em operações Cognito.
- Alerta para eventos de reconciliação entre Cognito e perfil interno.

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| Amazon Cognito User Pools | Gerenciado AWS | Fonte de verdade das credenciais e tokens |
| Identity Service | 1.0 | Implementa os endpoints `/v1/identity` |
| Aurora PostgreSQL Serverless v2, schema `identity` | PostgreSQL compatível com Aurora | Fonte do perfil interno |
| Amazon API Gateway | Gerenciado AWS | Entrada REST pública |
| AWS Secrets Manager | Gerenciado AWS | Armazena segredos operacionais |
| AWS KMS | Gerenciado AWS | Criptografia em repouso |
| Amazon CloudWatch Logs | Gerenciado AWS | Logs estruturados e Logs Insights |
| AWS X-Ray e OpenTelemetry | Gerenciado AWS e SDKs compatíveis | Tracing distribuído |

**Garantias de compatibilidade**

- Todos os endpoints ficam sob `/v1`.
- Alterações incompatíveis exigem nova versão de API.
- Tokens não são retornados no corpo das respostas.
- Cookies de sessão mantêm nomes estáveis: `cc_at` e `cc_rt`.
- O endpoint `GET /v1/identity/session/current` retorna somente identidade e
  estado da sessão, sem papéis por casa.
- A consulta de papéis por casa será responsabilidade do Household Service.

---

### 9. Critérios de aceite técnicos

- `POST /v1/identity/users` cria usuário no Cognito e perfil interno no schema
  `identity`.
- Cadastro com e-mail duplicado retorna `409`.
- Cadastro com senha fora da política retorna `422`.
- Senha com menos de 8 caracteres é rejeitada.
- Senha sem letra ou sem número é rejeitada.
- O usuário consegue fazer login imediatamente após cadastro.
- Login com credenciais válidas retorna `200` e grava cookies `HttpOnly`,
  `Secure` e `SameSite`.
- Login não retorna access token nem refresh token no corpo da resposta.
- Login com credenciais inválidas retorna `401` sem revelar se o e-mail existe.
- O sistema bloqueia login por `15 minutos` após `10` tentativas inválidas.
- O usuário só recebe aviso de limite nas `3` últimas tentativas.
- `GET /v1/identity/session/current` retorna `200` para sessão válida.
- `GET /v1/identity/session/current` renova a sessão quando access token estiver
  expirado e refresh token válido.
- `GET /v1/identity/session/current` retorna `401` e limpa cookies quando a
  sessão não for renovável.
- `DELETE /v1/identity/sessions/current` remove cookies e tenta revogar tokens
  no Cognito.
- O frontend consegue lembrar apenas o e-mail do usuário, nunca a senha.
- Logs não registram senha, access token, refresh token ou ID token.
- Métricas de cadastro, login, bloqueio, renovação, logout, latência e erros são
  emitidas.
- Spans de tracing são gerados para request, rate limit, Cognito, persistência e
  escrita de cookie.
- A feature atende `p95 < 150 ms` para todas as operações em condições normais
  de operação.

---

### 10. Riscos e mitigação

### Exposição de tokens ou credenciais no frontend

- **Probabilidade:** baixa
- **Impacto:** comprometimento de sessão e acesso indevido a dados financeiros.
- **Mitigação:**
    - Usar cookies `HttpOnly`, `Secure` e `SameSite`.
    - Nunca retornar tokens no corpo da resposta.
    - Proibir armazenamento de senha no navegador.
    - Revisar logs para impedir registro de tokens e credenciais.
- **Plano de contingência:** revogar sessões afetadas no Cognito, limpar cookies
  e investigar os logs de autenticação.

### Brute force contra login

- **Probabilidade:** média
- **Impacto:** tentativa de comprometimento de contas e degradação da
  disponibilidade.
- **Mitigação:**
    - Limitar login a `10` tentativas inválidas em `15 minutos`.
    - Bloquear novas tentativas por `15 minutos`.
    - Usar HMAC do e-mail normalizado para controle sem expor dados sensíveis.
    - Alertar sobre aumento anormal de falhas e bloqueios.
- **Plano de contingência:** elevar regras de proteção na borda e endurecer a
  política de bloqueio temporariamente.

### Divergência entre usuário no Cognito e perfil interno

- **Probabilidade:** média
- **Impacto:** usuário autenticável sem perfil interno ou perfil interno sem
  principal autenticável confiável.
- **Mitigação:**
    - Criar perfil interno logo após criação no Cognito.
    - Usar idempotência no cadastro.
    - Compensar somente principal comprovado, nunca após commit ambíguo.
    - Registrar evento de reconciliação quando a compensação falhar.
- **Plano de contingência:** bloquear acesso do usuário divergente e executar
  reconciliação operacional.

### Falha de renovação de sessão causando logout indevido

- **Probabilidade:** média
- **Impacto:** interrupção da experiência do usuário e possível perda de fluxo
  de trabalho em andamento.
- **Mitigação:**
    - Renovar sessão via `GET /v1/identity/session/current`.
    - Usar refresh token de `7 dias`.
    - Registrar falhas de renovação com `failureReason`.
    - Garantir que o frontend preserve estado de formulário quando possível.
- **Plano de contingência:** solicitar novo login e restaurar a navegação para o
  estado anterior quando tecnicamente possível.

### Acoplamento futuro se papéis por casa forem colocados no token

- **Probabilidade:** baixa
- **Impacto:** tokens ficariam desatualizados quando papéis mudassem e a
  autorização por casa ficaria mais difícil de manter.
- **Mitigação:**
    - Manter papéis por casa fora do token no MVP.
    - Consultar papéis pelo Household Service ao carregar a tela.
    - Centralizar autorização de casa e papel nos domínios responsáveis.
- **Plano de contingência:** manter tokens focados em identidade e migrar
  qualquer autorização indevida para consultas por casa.
