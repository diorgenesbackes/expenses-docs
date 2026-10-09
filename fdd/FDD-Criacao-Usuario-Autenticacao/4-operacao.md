# 4 — Operação: cadastro

Atualizado em 2026-10-07. Escopo deste guia: `POST /v1/identity/users`; login e sessões são descritos na [entrega E3/E4](2-desenvolvimento.md#login-e-sessoes-e3-e4).

## Implantação e reversão

1. Configurar conexão privada e Cognito conforme o [README do backend](../../../expenses-service/README.md). Nunca colocar senha, Client Secret ou credenciais AWS no frontend ou nos arquivos versionados.
2. Consultar `status --verbose` e aplicar o master Liquibase antes de iniciar a versão nova da API, conforme o [guia de migrações](../../../expenses-liquibase/README.md). O changeset novo é `sprint-2-001`; os oito anteriores não foram modificados. Os testes aplicam migrações somente em bancos descartáveis: não atualizam automaticamente o banco local `expenses`.
3. Usar HTTPS e mesma origem no navegador. Se necessário, configurar `Identity:AllowedOrigins` com origens exatas confiáveis; isso não configura CORS. O cadastro aceita somente JSON, verifica Origin quando presente e não usa cookies para autenticar. Cookies/CSRF de sessões pertencem à próxima entrega.
4. Verificar health e um cadastro sintético controlado. `GET /health/db` comprova conexão, não atualização do schema. `201` confirma conta e perfil; não inicia sessão. Guardar o UUID de idempotência no cliente antes do envio.
5. Em reversão da aplicação, preservar a tabela nova e seu histórico. O rollback SQL da sprint-2 remove o diário operacional e perde evidências de recuperação: usar somente em banco descartável ou após plano explícito de preservação e ausência de operações pendentes. Não desfazer a sprint-1 para reverter esta entrega.

## Repetição e estados

| Situação | Ação |
| --- | --- |
| Resposta perdida | Reenviar mesma chave UUID e mesmo e-mail; após conclusão retorna mesmo UUID/data. Senha diferente não modifica a conta. |
| `409 IDEMPOTENCY_REQUEST_IN_PROGRESS` | Aguardar a requisição original. Evitar repetição automática sem limite. |
| `409 IDEMPOTENCY_KEY_REUSED` | Corrigir a associação entre chave e intenção; nunca reutilizar chave para outro e-mail. |
| `409 IDEMPOTENCY_KEY_EXPIRED` | O resultado terminal não é mais recuperado após 24 h da reserva. Nova chave não contorna duplicidade de conta. |
| `409 IDENTITY_USER_EXISTS` | Pode ser conta existente ou reserva ativa de outra operação. Não redefinir senha nem excluir principal. |
| `503 IDENTITY_RECONCILIATION_REQUIRED` | Investigar antes de qualquer nova escrita. Manter a reserva bloqueada. |
| Falha conhecida com compensação concluída | Mesma chave reproduz a falha; uma nova tentativa pode usar nova chave. |

Etapas: `reserved`, `principal_created`, `password_ready`, `completed`, `failed`, `reconciliation_required`. Estados ativos mantêm unicidade do e-mail. Uma reserva com mais de cinco segundos é tratada como incerta no replay, inclusive após interrupção do processo. Expiração não remove operações incertas. Não há job de retenção ou reconciliação.

## Investigação de resultado incerto

O suporte usa `correlationId` retornado no header/ProblemDetails e `operationId` do evento de reconciliação. Logs não devem conter senha, tokens nem payload AWS. Consultar por parâmetro, sem copiar dados pessoais para tickets públicos:

```sql
SELECT id, correlation_id, stage, created_at, updated_at, version,
       user_id, cognito_username, cognito_sub, idempotency_record_id
FROM identity.registration_operations
WHERE correlation_id = :correlation_id;

SELECT operation_key, reason_code, created_at
FROM identity.identity_reconciliations
WHERE operation_key = :operation_id;
```

1. Confirmar ambiente/pool e que não existe requisição ainda em execução. Correlacionar diário, auditoria e perfil usando IDs. Cinco segundos é o orçamento cooperativo de processamento; não é prova isolada de que uma escrita remota não aconteceu.
2. Se a operação está `completed` e o perfil está vinculado ao mesmo sub, preservar o usuário. Recuperar o resultado pela chave válida. Nunca excluir principal após resposta de commit incerta.
3. Quando há username/sub comprovadamente devolvidos pela criação da operação, consultar o Cognito por username e conferir o sub. Se houver divergência, manter bloqueio e investigar; não alterar senha.
4. Quando não há identidade comprovada, encontrar um usuário pelo e-mail **não comprova propriedade**. Preservar a conta e buscar evidência adicional da operação no provedor. Não fazer exclusão ou redefinição por correspondência de e-mail.
5. Decidir entre finalizar perfil/resultados de forma atômica ou compensar o principal comprovado. Toda correção deve preservar auditoria e idempotência, usar bloqueio/versão da operação e ser revisada antes de executar. Esta versão não fornece comando administrativo automático; não editar apenas `stage`, remover a reserva ou marcar reconciliação concluída para desbloquear um e-mail.
6. Se o banco estava indisponível durante a falha, o evento operacional pode ser a única evidência adicional à reserva original. Investigar também reservas antigas: ausência de linha em identity_reconciliations não significa ausência de problema.

A API nunca continua o negócio em segundo plano após responder. Compensação usa apenas o tempo restante dos cinco segundos; quando não termina, mantém resultado incerto.

## Telemetria e evidências

- Logs JSON incluem correlação, trace W3C, requestId e encerramento HTTP; dados estruturados estão em State/Scopes do logger. Eventos de reconciliação incluem somente IDs operacionais. Exportação/normalização no coletor, alertas e dashboards continuam pendentes em E6.
- Meter/ActivitySource `Expenses.Identity.Registration`: `expenses.identity.user.created`, `expenses.identity.profile.reconciliation.needed` e `expenses.identity.registration.duration` (segundos, tag outcome). Adaptador externo: `Expenses.Identity.Cognito`.
- As métricas descrevem execuções observadas no processo; interrupções abruptas exigem consulta do diário para detectar reservas antigas.
- [Medição local](3-testes-latencia-cadastro.json): 20 cadastros HTTP sequenciais, TestServer + PostgreSQL local + Cognito real, p95 1202,96 ms. Meta aprovada de 150 ms não atendida. Medir na implantação de destino com carga e ingresso reais antes da homologação, sem excluir tempo de dependências nem antecipar resposta.

## 2026-10-08 — Operação da interface E5

O [README do frontend](../../../expenses-ui/README.md) descreve rotas, sessão e execução Vite/HTTPS. O bundle usa exclusivamente `/api/` na mesma origem e não contém configuração privada Cognito. O Compose HTTP atual não é suficiente para login real; configurar ingresso HTTPS e segredos no backend antes de liberar esse uso. Não desabilitar Secure, CSRF ou verificação TLS para contornar a configuração.

Para regressão local sem AWS, usar o laboratório descartável de expenses-tests. Os serviços de desenvolvimento foram parados nesta entrega e o banco existente foi preservado. Para voltar à interface anterior, reverter os fontes E5 e reconstruir o frontend; não há rollback de schema ou do backend nesta etapa. Não usar arquivos antigos de `dist` como fonte: o Dockerfile gera o bundle com `npm run build`.
