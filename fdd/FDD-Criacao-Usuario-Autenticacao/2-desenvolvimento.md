# 2 — Desenvolvimento: criação de usuário e autenticação

Histórico de implementação da FDD, separado por entrega. Consolidação documental em 2026-10-08; datas, resultados e pendências abaixo pertencem às entregas originais. Não representa nova implementação ou execução de testes.

[Especificação](FDD-Criacao-Usuario-Autenticacao.md) · [Planejamento](1-planejamento.md) · [Testes](3-testes.md) · [Operação](4-operacao.md)

## Cadastro E2

Data: 2026-10-07. Status: cadastro implementado e validado funcionalmente; aceite de desempenho pendente.

Referências: [plano da feature](1-planejamento.md), [FDD](FDD-Criacao-Usuario-Autenticacao.md), [contratos HTTP](../../guidelines/API-SINCRONA.md).

### Resultado esperado

`POST /v1/identity/users` deve criar uma conta autenticável no Cognito e um perfil interno no PostgreSQL. O sucesso só é devolvido após confirmação da persistência local. O cadastro não inicia uma sessão nem cria casa; o usuário fará login em seguida.

A integração externa já passou em teste real. Esta entrega conecta essa capacidade às regras de aplicação, banco e HTTP. Login/cookies, bloqueio de tentativas e revogação imediata são as entregas seguintes.

### Contrato implementado

- Request JSON com `email` e `password`, limitado a 8 KB.
- Validar formato e limites antes de chamar o Cognito. Normalizar e-mail com remoção de espaços externos e minúsculas, alinhando banco e provedor. Não alterar, cortar ou normalizar a senha.
- E-mail ASCII normalizado com até 128 caracteres. Senha de 8 a 256 caracteres, com pelo menos uma letra e um algarismo ASCII; espaços e caracteres de controle são rejeitados. O DTO limita o e-mail bruto a 320 caracteres antes da normalização.
- `Idempotency-Key`: UUID aleatório recomendado, conforme a FDD. O futuro frontend sempre deve fornecê-lo. Se ausente, manter proteção de unicidade, mas documentar que não há recuperação do resultado por chave após perda da resposta.
- Resposta `201`: `{ user: { id, email, createdAt } }`, com UUID e data UTC. Não retornar senha, tokens ou `cognitoSub`. Não anunciar Location para uma consulta individual inexistente.
- `Cache-Control: no-store` em sucesso/erro; correlação validada ou gerada pela API; erros ProblemDetails com código estável e identificadores de suporte.
- Controller separado da listagem de diagnóstico para permitir cadastro fora de Development sem publicar a listagem global.

| Situação | Resposta prevista |
| --- | --- |
| Payload ou header malformado | 400 |
| Corpo acima do limite | 413 |
| Política de senha não atendida | 422 `IDENTITY_PASSWORD_POLICY_VIOLATION` |
| E-mail já cadastrado, outra operação | 409 `IDENTITY_USER_EXISTS` |
| Mesma chave, outro e-mail | 409 `IDEMPOTENCY_KEY_REUSED` |
| Mesma operação ainda em execução | 409 `IDEMPOTENCY_REQUEST_IN_PROGRESS` |
| Mesma operação concluída | Repetir status e resposta seguros originais |
| Dependência indisponível ou timeout | 503, sem afirmar que a conta não foi criada |
| Falha interna | 500 `UNEXPECTED_ERROR` |
| Resultado incerto/reconciliação necessária | 503 `IDENTITY_RECONCILIATION_REQUIRED` |
| Origem de navegador não permitida | 403 |
| Content-Type incompatível | 415 |

### Semântica de repetição

Comportamento aprovado: recuperar o resultado de uma operação já concluída; impedir uma segunda execução enquanto a primeira está em andamento; distinguir outro cadastro com e-mail duplicado.

Semântica implementada:

- Escopo de chave por operação pública de cadastro. Vincular ao e-mail normalizado para verificar reutilização, sem autenticação prévia, pois o cadastro é anônimo.
- Revalidar o e-mail de entrada no replay. A chave não concede sessão nem acesso a dados adicionais do usuário.
- Guardar somente fingerprint dos campos não secretos, resposta pública e metadados operacionais. Nunca guardar senha ou derivação da senha no fingerprint.
- Uma alteração de senha em repetição da mesma chave não troca a credencial criada: reapresentar o resultado original. Uma nova intenção de cadastro usa nova chave; conta existente continua sendo conflito.
- Janela de replay de 24 horas a partir da reserva, definida no código. Chave terminal expirada retorna `409 IDEMPOTENCY_KEY_EXPIRED`; não há job de exclusão. Expirar uma chave não autoriza recriar conta existente.
- Operações de resultado incerto não devem ser descartadas nem reexecutadas automaticamente por vencimento de prazo. Precisam de reconciliação explícita.

### Persistência e recuperação

Reutilizar `identity.users`, `idempotency_records`, `identity_reconciliations` e `audit_logs`. Acrescentar via novo changeset Liquibase o estado operacional que hoje falta para cadastro. Preservar os changesets aplicados.

A estrutura adicional deve registrar a operação, reserva do e-mail normalizado, etapa, vínculo à idempotência (chave interna gerada quando o cliente a omite), username imutável/sub quando conhecidos, instantes e versão de concorrência. Não deve registrar senha, tokens ou payload de fornecedor. Uma restrição deve impedir duas operações ativas para o mesmo e-mail.

Etapas persistidas: `reserved` → `principal_created` → `password_ready` → `completed`. A reserva já cobre o intervalo de criação externa sem resposta. Falhas distinguem `failed` (resultado conhecido) e `reconciliation_required`. Transições usam atualização condicional/versão, nunca apenas consulta seguida de escrita sem proteção.

Fluxo:

1. Validar entrada; em transação curta, reservar a operação/e-mail e verificar replay ou conflito.
2. Confirmar a reserva antes de iniciar qualquer escrita externa.
3. Criar o principal Cognito com mensagens suprimidas. Persistir o username/sub devolvido antes da próxima etapa sempre que possível.
4. Definir senha permanente no principal que esta operação criou.
5. Em uma transação local, gravar perfil, auditoria, conclusão da operação e resultado de idempotência.
6. Responder somente depois do commit confirmado.

Nenhuma transação/lock do PostgreSQL permanece aberto durante chamadas ao Cognito. Cada operação externa preserva o limite de três segundos, mas todos os passos do caso de uso compartilham orçamento de cinco segundos, iniciado após leitura/validação HTTP do corpo. O limite não inclui upload, transporte nem espera anterior no servidor. Só iniciar compensação se houver orçamento; registrar reconciliação caso não seja possível concluí-la. Não adicionar trabalho em segundo plano após a resposta.

Se a resposta do commit se perder, consultar o resultado persistido antes de concluir que houve falha. Nunca compensar apagando o principal quando o perfil pode ter sido confirmado. Se o banco estiver indisponível, conservar estado incerto e emitir evento operacional seguro.

Se a criação externa responder com timeout, não presumir que falhou. Consultar um usuário por e-mail não comprova que ele foi criado por esta operação; não alterar senha nem excluir esse usuário automaticamente. Estado incerto exige reconciliação operacional, com procedimento documentado. Sem marcador confiável no provedor não prometer recuperação automática de todos os intervalos de falha.

### Entregas revisáveis

| Ordem | Trabalho | Evidência de conclusão |
| --- | --- | --- |
| 1 | Fechar contrato e corrigir FDD/OpenAPI | Exemplos de requests, erros e repetição consistentes |
| 2 | Migration e mapeamentos de cadastro | Aplicação/reaplicação/rollback em banco descartável; unicidade e concorrência verificadas |
| 3 | Caso de uso e repositórios | Sucesso, replay, falhas parciais e compensação com provedor simulado |
| 4 | Endpoint e correlação | Testes HTTP e disponibilidade fora de Development; listagem preservada |
| 5 | Integração completa | Cadastro HTTP com banco isolado e Cognito de desenvolvimento, limpeza confirmada |
| 6 | Medição e documentação | Tempos por operação, procedimento de reconciliação e limitações reportadas |

### Cenários obrigatórios

- E-mail inválido, senha curta/sem letra/sem número e payload acima de 8 KB não chamam o provedor.
- Normalizações equivalentes do e-mail não geram duas contas.
- Requisições simultâneas com a mesma chave chamam criação externa uma única vez.
- Chaves diferentes para o mesmo e-mail não causam redefinição de senha ou exclusão de conta existente.
- Resposta perdida após commit é recuperada com a mesma chave e mesmo UUID.
- Interrupção antes/depois da criação e antes/depois do commit mantém estado recuperável ou explicitamente incerto.
- Falha ao definir senha e falha ao salvar perfil exercitam compensação somente com propriedade comprovada.
- Falha na compensação/banco gera evento sanitizado e nenhum sucesso falso.
- Logs, auditoria, idempotência e respostas não contêm senha ou tokens sintéticos.
- Teste real registra um usuário, verifica vínculo por sub e login no Cognito e remove somente os dados sintéticos que criou.
- Todas as evidências distinguem teste simulado, PostgreSQL real e Cognito real.

### Sequência após E2

E3/E4 devem formar uma entrega de segurança coerente: não publicar login como pronto antes de cookies/CSRF, validação de tokens, sessão persistente e revogação imediata. Bloqueio será exclusivamente por e-mail; renovar sem sucesso encerra a sessão conforme decisão aprovada.

A meta p95 < 150 ms vale para todas as operações, incluindo as chamadas externas. Medir primeiro na topologia de destino, com carga e ponto de medição documentados. Uma amostra de SDK bem-sucedida não comprova desempenho do endpoint. Se o cadastro síncrono exceder a meta, registrar o impedimento e discutir arquitetura/requisito com o responsável; não ocultar a latência removendo chamadas da medição ou respondendo antes de concluir.

### Evidências da implementação — 2026-10-07

- Novo changeset `sprint-2-001`: `identity.registration_operations`; total de nove changesets e 34 tabelas. Validadas aplicação incremental preservando usuário existente, reaplicação, constraints, rollback da sprint-2 e ciclo completo em banco descartável.
- Suíte integrada inicial: **78 testes aprovados**, incluindo PostgreSQL e Cognito reais. Vinte cadastros HTTP sintéticos foram autenticados no Cognito, repetidos por chave e removidos com confirmação. O banco `expenses` não recebeu dados nem migrações desses testes.
- Medição exploratória: 20 amostras sequenciais, API em TestServer, PostgreSQL local e Cognito `us-east-1`, SDK previamente inicializado; **p95 1202,96 ms**, acima dos 150 ms. [Dados e metodologia](3-testes-latencia-cadastro.json). Não inclui ingresso/TLS de produção nem carga concorrente; não representa homologação.
- [Procedimento operacional](4-operacao.md) cobre implantação, investigação, reconciliação e reversão. Não há reconciliador automático ou endpoint administrativo nesta entrega.
- Métricas e atividades locais usam `Expenses.Identity.Registration`; JSON de logs inclui contexto em State/Scopes. Exportação, normalização no coletor, dashboards e alertas ficam em E6.

A entrega funcional E2 não encerra a FDD. E3/E4 foram implementadas posteriormente, conforme [entrega de sessões](2-desenvolvimento.md#login-e-sessoes-e3-e4); E5 (interface) e homologação continuam pendentes. A meta de desempenho permanece inalterada e não atendida nesta medição.

Revisão final: 78 testes aprovados com PostgreSQL real e dois testes AWS opt-in ignorados nessa rodada; acrescentados cenários de rollback atômico em falha de auditoria e e-mail com domínio inválido. Após o último ajuste de middleware, os 11 testes HTTP de cadastro passaram, incluindo proteção de origem na rota com barra final.

<a id="login-e-sessoes-e3-e4"></a>

## Login e sessões E3/E4

Implementação: 2026-10-07. Revisão final: 2026-10-08. Implementação backend; interface React e homologação de desempenho continuam pendentes.

### Contrato HTTP

| Método/rota | Resultado | Proteção |
| --- | --- | --- |
| `GET /v1/identity/csrf` | `200 { token }`, cookie `cc_csrf` | HTTPS, no-store; token CSRF não autentica. |
| `POST /v1/identity/sessions` | `200 { user, session }`, cookies de sessão | JSON email/password, HTTPS, cookie CSRF + `X-CSRF-Token`, origem permitida quando presente. |
| `GET /v1/identity/session/current` | `200 { user, session }`; pode renovar cookies | Cookies de sessão e CSRF, inclusive no GET. |
| `DELETE /v1/identity/sessions/current` | `204` sem corpo após revogação local | Cookies de sessão e CSRF; repetição da sessão já revogada é idempotente. |

`user` contém somente UUID e e-mail. `session` contém `authenticated`, `refreshed`, `accessTokenExpiresInSeconds` e `refreshTokenExpiresInSeconds`. Não há senha/token JWT no JSON. Todos usam `Cache-Control: no-store`, correlação UUID e trace W3C. Login não aceita idempotência: cada login bem-sucedido cria uma sessão independente, sem encerrar sessões de outros dispositivos. Repetir login pode criar outra sessão; a interface deve impedir envio duplicado.

No navegador: buscar CSRF, guardar o token apenas em memória e enviar `X-CSRF-Token` nas três operações. O cookie CSRF é HttpOnly. Não guardar credenciais no localStorage. Depois de recarregar a página, buscar novo token CSRF antes de consultar a sessão. Origens confiáveis adicionais são exatas em `Identity:AllowedOrigins`; não há CORS permissivo.

| Situação | HTTP/código |
| --- | --- |
| Entrada inválida | `400 REQUEST_VALIDATION_FAILED` |
| Credencial/sessão inválida ou expirada | `401 IDENTITY_INVALID_CREDENTIALS`; desafio de rota protegida: `IDENTITY_SESSION_INVALID` |
| CSRF ausente/inválido, origem não permitida ou sessão via HTTP | `403 IDENTITY_CSRF_INVALID` (o middleware de login pode usar `RESOURCE_ACCESS_DENIED` para origem) |
| Operação concorrente | `409 IDENTITY_OPERATION_IN_PROGRESS`, `Retry-After` |
| Login bloqueado | `423 IDENTITY_LOGIN_BLOCKED`, `Retry-After` em segundos |
| Dependência indisponível | `503 DEPENDENCY_UNAVAILABLE` |
| Perfil ausente após autenticação válida | `503 IDENTITY_RECONCILIATION_REQUIRED`, sem cookies de sessão |
| Corpo de login acima de 8 KB/formato incompatível | `413`/`415` |

O caso de uso tem prazo cooperativo de cinco segundos, após leitura/validação HTTP; cada chamada Cognito tem limite de três segundos e não possui retry automático. Transporte, upload e espera anterior não estão nessa medida. O cadastro conserva seu contrato E2.

### Bloqueio exclusivamente por e-mail

- Normalizar e-mail para minúsculas e remover espaços externos; mesmo contrato ASCII/128 caracteres do cadastro.
- Identificar o e-mail no controle de tentativas por HMAC-SHA256 com chave privada estável. IP não participa do bloqueio nem é coletado nesse fluxo.
- Contar somente credenciais inválidas em janela móvel `(agora - 15 min, agora]`. Na sétima, oitava e nona falhas retornar `remainingAttempts` igual a 3, 2 e 1. A décima falha inicia bloqueio de 15 minutos e retorna `423`.
- Requisições bloqueadas não chamam Cognito e não estendem o prazo. Autenticação bem-sucedida remove tentativas/bloqueio desse e-mail. Falha operacional não conta como senha incorreta.
- Uma reserva por e-mail em `login_guards` serializa chamadas concorrentes entre réplicas, sem manter transação durante Cognito. Concorrente recebe `409`, sem contar tentativa. Reserva abandonada expira em dez segundos; conclusão exige o mesmo proprietário e reserva válida.
- Proteções nativas do Cognito continuam aplicáveis; não são substituídas por essa política.

### Sessão e garantias de segurança

`cc_at` carrega access token; `cc_rt` carrega refresh token e UUID da sessão em um envelope autenticado e criptografado por ASP.NET Data Protection. Ambos são HttpOnly, Secure, SameSite=Strict, host-only, Path=/; exclusão usa os mesmos atributos e respeita cookies fragmentados. Refresh possui prazo absoluto local de sete dias, sem prorrogação em renovações. O pool deve manter a configuração aprovada de access token de 15 minutos e refresh de sete dias.

O banco armazena somente hashes SHA-256 dos tokens, vínculo ao perfil, datas e estado. Credenciais não são registradas em logs, auditoria ou idempotência. A API não aceita um bearer JWT isolado: exige o ticket protegido e o vínculo persistente da sessão.

O validador usa Microsoft IdentityModel, assinatura RS256, issuer do pool, `client_id`, `token_use=access`, subject e expiração sem tolerância adicional. Chaves públicas vêm da descoberta OIDC/JWKS HTTPS e usam cache/atualização do ConfigurationManager. `kid` desconhecido solicita atualização; a requisição é rejeitada sem aceitar assinatura desconhecida. Erros do fornecedor permanecem sanitizados.

Rotas futuras protegidas usam `[Authorize]`, cuja política padrão exige `ExpensesSession`. Cada autorização valida o JWT e depois lê a sessão no PostgreSQL, sem cache positivo. Esse último acesso ao banco é o ponto de autorização: revogação já confirmada impede novas autorizações. Requisições autorizadas antes do commit do logout podem terminar; não há cancelamento retroativo de operações em andamento. Escritas autenticadas também devem aplicar `SessionCsrfFilter` e sua autorização por recurso.

Renovação disputa uma transição condicional `active → refreshing`. Durante a chamada externa, a sessão não autoriza acesso protegido. Uma segunda consulta em disputa retorna `409` sem limpar cookies. Sucesso só restaura `active` se o proprietário da renovação ainda for o mesmo. O logout pode mudar qualquer estado para `revoked`; assim uma renovação tardia nunca o desfaz.

Qualquer falha após iniciar renovação mantém a sessão sem autorização, tenta marcar `revoked`, limpa cookies e retorna `401` ou `503`. Se até a gravação de falha não for possível, `refreshing` já é persistente e nega acesso. Uma renovação abandonada por mais de cinco segundos exige novo login em vez de tentar reexecutar a chamada externa. Não há job nem fila.

Logout confirma revogação e auditoria em uma transação local antes de tentar a revogação Cognito. Falha remota não desfaz revogação local; emite contador operacional. Falha na persistência local retorna `503`, nunca `204`; cookies são removidos, mas a API não promete revogação persistente nessa resposta. Sem acesso ao banco, autorização falha fechada. Não há garantia de revogação de JWT em serviços externos que ignorem o registro de sessão desta API.

### Configuração e operação

1. Aplicar master Liquibase até `sprint-3-001` antes de publicar a API. São dez changesets e 36 tabelas. A migration cria `identity.sessions`, `identity.login_guards` e permite `ip_hash` nulo nas tentativas antigas; não altera checksums anteriores.
2. Configurar `Identity:EmailHmacKey` em armazenamento privado: Base64 de pelo menos 32 bytes aleatórios, compartilhado entre réplicas. Localmente, a partir de `expenses-infrastructure/`, `python3 scripts/configure-session-secret.py` gera a chave sem exibir nem sobrescrever a existente. Rotação exige plano para preservar bloqueios ativos; trocar a chave silenciosamente reinicia a identidade dos contadores.
3. Preservar as chaves de Data Protection com application name `Expenses.Identity`. No desenvolvimento, usa-se o key ring padrão persistido pelo ASP.NET. Para implantação, todas as réplicas precisam de um key ring durável compartilhado e protegido em repouso; provisionamento do cofre/certificado pertence à infraestrutura de destino. Não publicar réplicas com chaves efêmeras ou distintas.
4. Usar HTTPS e mesma origem. Proxy reverso precisa encaminhar o esquema correto por configuração explícita de proxies confiáveis; não confiar indiscriminadamente em headers forwarded. A configuração de produção desse ingresso ainda precisa ser homologada.
5. Não excluir sessões revogadas ou reservas para resolver uma ocorrência. Consultar IDs e correlação sem imprimir cookies/tokens. `refreshing` antigo exige novo login; perfil ausente exige reconciliação por sub, nunca vinculação automática apenas pelo e-mail.
6. Reversão da aplicação deve preservar tabelas de sessão e evidências. Rollback da sprint-3 descarta sessões/guards; somente em banco descartável ou manutenção planejada. Por preservação de histórico, deixa `ip_hash` nullable: não fabrica IP nem elimina tentativas antigas para restaurar NOT NULL. Reaplicação foi testada.

Não há limpeza automática de tentativas, sessões e guards. Definir retenção operacional em E6 com preservação dos bloqueios e evidências; não adicionar jobs de negócio. Configuração local de segredo foi concluída nesta entrega. As migrações de teste foram aplicadas apenas em bancos descartáveis, sem alterar `expenses`.

### Validação e limitações

Rodada integrada final em 2026-10-07: **98 testes aprovados, zero falhas e zero ignorados**, incluindo os três testes opt-in de Cognito (SDK, cadastro HTTP e sessões HTTP). Build concluído sem avisos de compilação. Contas sintéticas removidas e banco descartável excluído. O validador Liquibase também passou: upgrade com perfil existente, reaplicação, integridade, rollback parcial/completo e reconstrução das 36 tabelas. Revisão final de diffs em 2026-10-08 sem erros de whitespace. As advertências de limite de corpo do TestServer são próprias do host de teste; o middleware aplica o limite de 8 KB também nesse host.

Testes cobrem JWT assinado (issuer, aplicação, tipo, assinatura, datas), cookies/CSRF, janela/bloqueio por e-mail, falhas operacionais, reservas concorrentes, renovação versus logout, expiração absoluta, falha na persistência da revogação e rejeição de cookies copiados entre instâncias. PostgreSQL real é usado para comprovar transações e concorrência.

O teste `Category=CognitoSessions` percorre cadastro, login HTTP, consulta, renovação com rotação, autorização e logout com Cognito real, removendo a conta sintética ao terminar. [Amostra de tempos](3-testes-latencia-sessoes.json): login 1255 ms, consulta 11 ms, refresh 341 ms, autorização 3 ms e logout 315 ms. Uma amostra por operação não mede p95; o login inclui descoberta de chaves fria. **Não declarar a meta de 150 ms atendida**: dependências externas ultrapassaram esse valor localmente. Homologar carga, topologia e ingresso de destino em E6.

Telemetria local: Meter/ActivitySource `Expenses.Identity.Sessions`, histogram `expenses.identity.session.duration` (segundos), contador `expenses.identity.session.operations` por operação/resultado e `expenses.identity.session.remote_revoke.failures`. Usar também telemetria do adaptador Cognito. Exportação, dashboards e alertas permanecem pendentes; não há IDs pessoais nos labels.

## 2026-10-08 — E5: interface React

- Substituído template por telas responsivas de login e cadastro, com identidade autenticada e logout. Rotas `/login`, `/cadastro` e `/`; links nativos com History API, sem dependência adicional de roteamento para estas três vistas.
- Cadastro não inicia sessão. Login normaliza e-mail; senha não é aparada. Validação visual acompanha o contrato; o backend permanece autoridade. Campos têm labels, foco no primeiro erro/resumo, senha revelável e proteção contra envio duplicado.
- Adaptador HTTP centraliza `/api/v1/identity`, credenciais same-origin, CSRF em memória, validação de respostas desconhecidas, timeout e mensagens por códigos estáveis. Sem retry automático de escrita/renovação. Repetição manual de cadastro no mesmo formulário/e-mail mantém chave de idempotência; não representa troca de senha.
- Consulta de sessão ao abrir, retomar foco/visibilidade e expirar acesso. Resultado atrasado de consulta não restaura estado anterior após login/logout. Conflito 409 preserva identidade; outras falhas removem a visão autenticada. Dados de autenticação não são persistidos pelo React.
- Lembrar e-mail depende de checkbox; desmarcar remove imediatamente. BroadcastChannel comunica login/saída entre abas sem transmitir dados pessoais ou credenciais. O backend continua garantindo revogação imediata, inclusive entre instâncias.
- Logout indisponível não mostra sucesso: remove identidade visível e informa que não foi possível confirmar revogação. Uma aba aberta volta a consultar o backend ao receber foco, inclusive sem BroadcastChannel.
- Vite ganhou proxy com validação TLS habilitada e suporte a caminhos privados de certificados. Nginx já tinha fallback de rotas; laboratório recebeu fallback restrito a `/`, `/login`, `/cadastro`, mantendo assets inválidos como 404.
- Nenhuma migration, chamada AWS, criação de conta real ou dependência de runtime adicionada. Frontend/backend de desenvolvimento parados; PostgreSQL pessoal preservado.

Unidades permanecem em `expenses-ui`; jornadas reais ficam em `expenses-tests`. Evidências e limitações no [relatório E5](../../testing/2026-10-08_17-12-26/2-execucao.md). O Compose existente ainda precisa de HTTPS e configuração privada para autenticação Cognito real; os testes E5 usaram o laboratório HTTPS com identidade simulada. A FDD inteira não está concluída.
