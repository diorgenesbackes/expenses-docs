# 1 — Planejamento: criação de usuário e autenticação

Criado em 2026-10-06. Atualizado em 2026-10-08. Status: planejamento em execução, com decisões aprovadas e integração Cognito validada. E2 e E3/E4 estão implementadas funcionalmente no backend; interface e homologação continuam futuras, salvo evidência explícita.

Base: [FDD](FDD-Criacao-Usuario-Autenticacao.md), [HLD](../../HLD.md), [backend](../../guidelines/BACKEND.md), [contratos HTTP](../../guidelines/API-SINCRONA.md) e [modelo físico](../../db/DB-Modelo-Fisico.md).

## Objetivo e limites

Entregar cadastro, login, consulta/renovação de sessão e logout no React e na API, com Cognito acessado apenas pelo backend. Manter o monólito modular síncrono. Household, convites, recuperação de senha, confirmação de e-mail, MFA e login social continuam fora desta entrega.

O cadastro termina com orientação para login, conforme a FDD. A tela autenticada inicial mostra identidade e saída; não antecipa o cadastro de casas.

## Ponto de partida

- API, Application, Domain, Infrastructure e testes já existem. Há cadastro público `POST /v1/identity/users`, health checks e listagem de diagnóstico apenas em Development; o cadastro integra Cognito e PostgreSQL.
- As seis tabelas de Identity existem nos scripts Liquibase: users, login_attempts, login_blocks, identity_reconciliations, idempotency_records e audit_logs. A sprint-2 acrescenta registration_operations; users e registration_operations estão mapeadas no EF Core e o restante da persistência do cadastro usa SQL parametrizado.
- A listagem de usuários é um diagnóstico restrito a Development. Separar o controller de cadastro para que a restrição atual não desabilite a nova funcionalidade em produção.
- React ainda é o template; faltam telas, navegação, integração e testes de interface.
- A configuração privada, o perfil AWS e as permissões usadas no ciclo Cognito foram validados por teste real. PostgreSQL local permite desenvolvimento sem provisionar Aurora inicialmente; homologação deve validar o ambiente de destino.

## Decisões da primeira entrega

Decisões confirmadas pelo responsável em 2026-10-06: bloqueio apenas por e-mail, encerramento da sessão em falha de renovação inclusive indisponibilidade, invalidação imediata no logout, p95 abaixo de 150 ms para todas as operações e IDs UUID. As decisões foram incorporadas à FDD v1.2; o contrato implementado de sessão está na entrega E3/E4.

| Tema | Encaminhamento |
| --- | --- |
| Identificador | Preservar UUID do banco e API atuais; corrigir exemplos `usr_...` da FDD. |
| Cadastro Cognito | Validar um fluxo administrativo com mensagens suprimidas e senha permanente, permitindo login imediato. Não marcar e-mail como verificado sem comprovação. Desabilitar cadastro direto que contorne o backend. |
| Cookies e CSRF | Definir SameSite, Path, domínio, duração e expiração dos cookies; preferir mesma origem e HTTPS local. Aplicar proteção contra CSRF, incluindo login e a consulta que renova sessão. Definir origens/proxies confiáveis e CORS conforme implantação. |
| Bloqueio | Confirmado: apenas por e-mail normalizado, protegido por HMAC. Não bloquear por IP. Especificar janela móvel, 10ª falha, avisos com 3/2/1 tentativas restantes, reset após sucesso e `Retry-After`. Persistência precisa coordenar requisições concorrentes entre réplicas. |
| Idempotência | Definir obrigatoriedade, escopo anônimo, validade e respostas para operação em andamento. Não persistir senha, tokens ou fingerprint derivado da senha. Explicitar que retry da mesma operação não altera credenciais e como tratar conteúdo divergente sem guardar senha. |
| Renovação | Confirmado: encerrar sessão e limpar cookies também em indisponibilidade. Manter distinção HTTP entre credencial inválida (`401`) e falha operacional (`503`). Definir rotação, chamadas simultâneas e prazo restante real do refresh. |
| Logout | Confirmado: invalidação imediata na API, inclusive de access tokens já copiados. Planejar registro persistente de sessão/revogação, verificado em cada requisição, e coordenação com refresh concorrente. Limpar cookies e tentar revogação no Cognito. Não confirmar revogação local se a persistência falhar; falhar fechado na validação de sessão. Harmonizar os status HTTP com essa garantia. |
| Falhas parciais | Definir recuperação após timeout ambíguo, interrupção do processo e indisponibilidade do banco. Compensar somente principal comprovadamente criado pela operação; nunca excluir conta preexistente. |
| Latência | Confirmado: p95 < 150 ms para todas as operações. Definir ponto de medição, carga e ambiente; medir separadamente cadastro, login, consulta, refresh e logout, incluindo dependências externas. É requisito a demonstrar, não garantia oferecida por configuração Cognito. Chamadas de até 3 s, retries e compensação devem caber no orçamento total de 5 s ou exigir revisão explícita da FDD. |

Revogação no Cognito não é detectada por uma validação JWT que apenas verifica assinatura e expiração. A exigência de invalidação imediata requer controle adicional persistente e nova migration Liquibase. Após confirmação da revogação local, novas validações devem rejeitar a sessão; requisições já autorizadas exigem semântica própria, a documentar. Não usar cache positivo que adie a revogação entre réplicas.

Idempotência aprovada em 2026-10-07: se o cadastro for concluído mas a resposta se perder, repetir a mesma operação deve recuperar o resultado sem criar outra conta. Mesma chave concluída retorna resultado original; em andamento retorna conflito temporário; outro cadastro com o mesmo e-mail retorna conflito de duplicidade. Não tratar repetição como troca de senha. Escopo, validade e comparação de conteúdo estão documentados na entrega E2.

Preparação AWS concluída para a prova de integração em 2026-10-07: configuração do pool informada pelo responsável, autenticação AWS local, segredo privado e operações do adaptador validados. O teste de ciclo não equivale a uma auditoria completa de todas as configurações do pool.

## Entregas em ordem de dependência

Ambiente validado em 2026-10-07: perfil local `expenses-dev` como `arn:aws:iam::633510959274:user/expenses-dev-local`; pool `us-east-1_DyDv5WCxr`; App Client `5fsk7rmh7tfej48nhmtuvkhbak`; região `us-east-1`. O adaptador consome a configuração de Development e o segredo dos User Secrets. O teste real confirmou as permissões das operações exercitadas, além da identidade confirmada via STS.

### E0 — Contratos e decisões

Concluir a tabela acima, atualizar exemplos HTTP/OpenAPI e montar a matriz de erros e cenários. Definir tratamento de body acima de 8 KB, correlação, cache desabilitado e campos seguros de resposta. `204` não tem corpo. Resolver os detalhes do Cognito em prova técnica isolada.

Aceite: contratos sem ambiguidades para frontend/backend; configuração necessária e limitações registradas.

### E1 — Fundação de Identity e ambiente de integração

- Definir portas consumidas pelos casos de uso para Cognito, persistência e relógio; SDK AWS e EF ficam em Infrastructure.
- Colocar contratos HTTP novos na API e entradas/resultados de casos de uso em Application; regras independentes em Domain.
- Mapear as tabelas de Identity necessárias e preparar transações curtas, sem locks de banco durante chamadas Cognito.
- Configurar tratamento de erros, correlação, limites, timeouts e instrumentação básica desde o primeiro fluxo.
- Preparar configuração privada, permissões AWS mínimas, HTTPS local, proxy React e validação de configuração.
- Usar novas migrations somente para lacunas comprovadas; preservar os changesets existentes.

Aceite: API inicia, diagnósticos continuam com o mesmo comportamento e integração pode ser substituída por um adaptador de teste. Simulação fica restrita aos testes; indisponibilidade em execução nunca autentica por fallback.

### E2 — Cadastro completo na API

Detalhamento executável: [entrega de cadastro E2](2-desenvolvimento.md#cadastro-e2). O documento registra o contrato implementado, recuperação, testes de aceite e pendência de desempenho.

Implementar `POST /v1/identity/users`: validação de e-mail e senha, normalização consistente, idempotência, criação no Cognito, persistência do perfil e resposta `201`. Tratar duplicidade, concorrência, perda de resposta após commit e compensação. Sincronização/reconciliação usa identidade confiável do Cognito, nunca apenas correspondência por e-mail.

Planejar registro operacional seguro quando nem o banco permite gravar reconciliação e procedimento manual para recuperar operações interrompidas; sem filas ou jobs de negócio.

Aceite: conta pode autenticar imediatamente; falhas não produzem falso sucesso nem exclusão de contas alheias; nenhum segredo é persistido na idempotência ou auditoria.

### E3 — Login e proteção de tentativas

Implementar `POST /v1/identity/sessions`, autenticação Cognito, resolução do perfil interno e cookies. Implementar HMAC de e-mail com segredo privado, controle de concorrência e expiração consultada durante a requisição. IP não participa do bloqueio; avaliar em nova migration a obrigatoriedade atual de ip_hash e sua eventual finalidade de auditoria. Falhas operacionais não contam como senha incorreta. Medir também interferência da proteção nativa do Cognito, que não é substituída pelo contador da API.

Aceite: login válido retorna perfil sem tokens; credencial incorreta usa mensagem genérica; avisos e bloqueio seguem os limites; testes concorrentes demonstram a política definida.

### E4 — Sessão, renovação e logout

Implementar `GET /v1/identity/session/current` e `DELETE /v1/identity/sessions/current`. Validar assinatura, emissor, aplicação cliente, tipo de token e expiração; carregar perfil vinculado ao sub. Definir cache/rotação de chaves de assinatura. Renovar conforme política definida e coordenar disputas entre abas/requisições. Aplicar cookies/CSRF e revogação conforme E0.

Aceite: sessão válida retorna identidade, expirada é renovada quando permitido, sessão inválida recebe `401`, indisponibilidade recebe tratamento distinto, logout limpa cookies com os mesmos atributos usados na criação.

### E5 — Fluxos React

Criar telas de cadastro e login, consulta inicial de sessão, área autenticada mínima e logout. Centralizar transporte HTTP, tratamento de erros e estado de sessão. Implementar lembrar somente e-mail mediante opção do usuário, remover preferência ao desmarcar, impedir envio duplicado e tratar bloqueio/indisponibilidade. Não persistir senha ou tokens no navegador. Evitar ciclos de retry/renovação e redirecionamentos externos não validados.

Adicionar infraestrutura de testes de interface, validando ferramentas/versões antes da instalação. Cobrir teclado, labels, foco, mensagens, telas pequenas, recarga, expiração e limpeza de dados privados no logout.

Aceite: usuário conclui cadastro → login → recarga com sessão → logout pelo navegador real, incluindo cookies Secure em HTTPS.

### E6 — Homologação e operação

Executar integração com Cognito real em ambiente isolado, PostgreSQL real com Liquibase e poucos fluxos E2E. Validar limites de cookies/cabeçalhos e encaminhamento de `Set-Cookie` na topologia de destino. Concluir métricas, traces, dashboards, alertas, documentação de configuração e procedimento de reconciliação/retenção.

Medir latência por operação e registrar resultado, carga e ambiente. Não declarar a meta de desempenho atendida sem medição. Documentar implantação, dependências e recuperação, preservando usuários/dados em reversões da aplicação.

Aceite: critérios da FDD associados a evidências; testes ignorados e pendências explícitos; nenhuma configuração falsa de autenticação habilitada no ambiente publicado.

## Estratégia de verificação

- Testes de regras com relógio controlável: senha, normalização, janela de 15 minutos, bloqueio e expiração.
- Testes HTTP com WebApplicationFactory: status, corpos, cabeçalhos, ausência de tokens, CSRF, payload, cancelamento e cookies.
- PostgreSQL isolado: unicidade, transações, idempotência concorrente, tentativas/bloqueios e reconciliação.
- Adaptador Cognito simulado: erro antes/depois de efeito externo, timeout ambíguo, compensação que falha e falha de persistência.
- Cognito real isolado: cadastro sem desafio obrigatório, login, refresh, revogação e comportamento de token revogado na API.
- React/E2E: fluxo principal, erros, lembrança de e-mail, múltiplas abas e sessão após recarga.
- Verificar logs, respostas e armazenamento usando segredos sintéticos para detectar vazamentos.
- Executar build/testes backend, lint/build frontend e validação Liquibase quando houver mudança de schema. Scripts frontend de teste serão adicionados junto das ferramentas; não existem hoje.

## Marcos e dependências externas

### Evidência de integração em 2026-10-07

A primeira parte de E1 está implementada: porta em Application, adaptador SDK AWS
v4 em Infrastructure, registro na API, configuração privada, prazo de três
segundos por operação, cancelamento, erros sanitizados e instrumentação local.
Não foram publicados endpoints de autenticação nesta entrega.

Validação: build e suíte local com 46 testes aprovados; dois testes externos
ignorados por padrão. O teste Cognito foi depois executado explicitamente e
aprovado no pool informado: criou usuário sintético, definiu senha permanente,
autenticou, renovou com rotação, revogou o refresh e confirmou a exclusão da conta.
O teste PostgreSQL não foi executado nesta entrega (nenhuma alteração de schema
ou persistência). Não houve medição de p95.

### Evidência E2 — 2026-10-07

Cadastro HTTP, perfil, idempotência, concorrência, compensação e registro de reconciliação implementados. A suíte integrada passou com 78 testes, incluindo 20 cadastros no Cognito real e PostgreSQL descartável. Migração incremental e rollback validados. Consulte [E2](2-desenvolvimento.md#cadastro-e2) e o [procedimento operacional](4-operacao.md).

M1 está funcionalmente demonstrado; o aceite de desempenho permanece pendente: p95 local de 1202,96 ms contra meta de 150 ms. Na conclusão isolada de E2, login e sessões ainda não estavam implementados. A evidência posterior de E3/E4 abaixo registra sua implementação; interface, exportação de telemetria e homologação continuam pendentes.

M1: E0–E2 entregam cadastro verificável pela API. M2: E3–E4 entregam ciclo completo de sessão. M3: E5–E6 entregam experiência no navegador homologada.

Cada entrega deve resultar em alteração revisável com comportamento, testes, impacto de migração e recuperação descritos. Não estimar prazo fechado antes de E0 e da verificação de acesso ao Cognito. Preparação da AWS requer confirmar ambiente, região, User Pool/App Client e permissões disponíveis, sem compartilhar segredos na conversa.

## Referências externas verificadas no planejamento

- [Criação administrativa de usuários e senha permanente](https://docs.aws.amazon.com/cognito/latest/developerguide/how-to-create-user-accounts.html).
- [AdminCreateUser e supressão de mensagens](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminCreateUser.html).
- [Renovação e rotação de refresh tokens](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-refresh-token.html).
- [Revogação e limites da validação JWT local](https://docs.aws.amazon.com/cognito/latest/developerguide/token-revocation.html).


### Evidência E3/E4 — 2026-10-07

Implementados login, cookies seguros, CSRF, sessão atual/renovação, bloqueio exclusivo
por e-mail e revogação persistente imediata. JWT é validado com chaves do Cognito;
autorização consulta PostgreSQL sem cache positivo. Renovação e logout coordenam
transições para impedir reativação da sessão revogada. [Contrato, evidências e limites](2-desenvolvimento.md#login-e-sessoes-e3-e4).

A migration sprint-3 acrescenta sessions e login_guards e permite tentativas sem IP.
Key ring compartilhado/protegido, ingresso HTTPS, exportação de telemetria, interface
E5 e homologação E6 continuam pendentes. A meta de 150 ms não foi demonstrada.

Revisão final E3/E4 em 2026-10-08: 98 testes integrados aprovados, sem ignorados, incluindo Cognito real; ciclo Liquibase aprovado. Nenhuma migration foi aplicada ao banco local expenses nesta validação.

## Atualização — 2026-10-08: E5

Interface de login/cadastro, identidade autenticada e saída implementada. Navegação nativa, transporte do módulo Identity, CSRF em memória e preferência explícita de e-mail. Validação local em [ciclo E5](../../testing/2026-10-08_17-12-26/1-plano.md). A etapa não antecipa Household nem remove as pendências de E6, HTTPS/configuração privada do Compose e desempenho real.
