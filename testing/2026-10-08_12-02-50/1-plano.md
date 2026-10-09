# 1 — Plano: testes locais

Registro documental: **2026-10-08 12:02:50 — America/Sao_Paulo**. Histórico migrado, preservando datas/IDs e resultados originais; esta data não indica nova execução.

[Padrão e Definition of Done](../README.md) · [Plano](1-plano.md) · [Execução](2-execucao.md) · [Histórico da FDD](../../fdd/FDD-Criacao-Usuario-Autenticacao/3-testes.md)

Data: 2026-10-08. Plano aprovado; implementação iniciada. Este documento preserva o escopo proposto. Para o que foi efetivamente entregue, executado e o que permanece pendente, consultar [Entrega dos testes locais](2-execucao.md) e o [runner](../../../expenses-tests/README.md).

## 1. Decisões de escopo

- Primeiro executar no ambiente local. A migração do laboratório e a homologação na nuvem ficam para uma etapa posterior; orçamento AWS não bloqueia esta fase.
- Backend e frontend mantêm somente seus testes unitários.
- Criar o projeto separado `expenses-tests`, irmão dos projetos existentes, para testes de integração, HTTP, navegador, carga/performance, recuperação e segurança.
- Preservar os testes já existentes, redistribuindo-os por responsabilidade. Não reescrever a suíte inteira apenas para mudar sua localização.
- Escala de produto informada: até 1.000 usuários cadastrados. Simultaneidade e taxa de requisições ainda são hipóteses a validar; não se deduz capacidade de produção desse número.
- Preservar requisito aprovado de p95 < 150 ms por operação. Resultado local com identidade simulada não comprova esse requisito com Cognito.

## 2. Inventário real encontrado

Contagem estática dos casos Fact e InlineData atuais, conferida com os 98 resultados da última execução integrada. Não foi executada novamente a suíte para este levantamento. A contagem inclui variações de teorias; não significa 98 jornadas independentes.

| Grupo principal | Casos | Arquivos/classes | Destino proposto |
| --- | ---: | --- | --- |
| Isolados, sem HTTP/banco/AWS reais | 46 | UserServiceTests (7), RegistrationServiceTests (16), CognitoIdentityProviderTests (15), CognitoAccessTokenValidatorTests (8) | Backend, `Expenses.UnitTests` |
| HTTP hospedado no processo | 35 | UsersEndpointTests (18), RegistrationEndpointTests (11), DatabaseHealthEndpointTests (6) | `expenses-tests/integration` |
| PostgreSQL real, parte com chamadas HTTP | 14 | UsersPostgresTests (1), RegistrationPostgresTests (5), SessionPostgresTests (8) | `expenses-tests/integration` |
| Cognito real, incluindo ciclos HTTP | 3 | CognitoLiveTests, RegistrationCognitoLiveTests, SessionCognitoLiveTests | `expenses-tests/external`, opt-in |
| Navegador/telas | 0 | Não há Playwright ou suíte equivalente | `expenses-tests/e2e`, após telas disponíveis |
| Carga/performance formal | 0 | Existem amostras de tempo, não um ensaio de capacidade | `expenses-tests/performance` |
| Varredura de segurança automatizada | 0 | Existem verificações de segurança dentro das suítes acima, não scanner abrangente | `expenses-tests/security` |

A classificação usa a dependência mais externa exercitada para não contar o mesmo caso duas vezes. Por exemplo, um teste HTTP com PostgreSQL está no grupo PostgreSQL. Os 46 isolados incluem adaptadores com SDK substituído e validação criptográfica com chaves geradas em memória; não dependem de conectividade AWS. A organização final deve manter a classificação por comportamento, não pelo nome da classe.

Outros achados:

- `expenses-ui` é o template React/Vite; não possui telas de cadastro/login, runner nem testes de interface. Não é possível declarar a jornada visual de autenticação pronta.
- WebApplicationFactory inicia a aplicação no processo de testes. É integração HTTP útil, mas não atravessa Kestrel publicado, proxy, certificado e rede entre containers.
- O Compose atual publica HTTP, usa containers com nomes fixos e o volume de desenvolvimento. Não deve receber carga, varreduras ativas ou limpeza de fixtures.
- Nginx já encaminha `/api/`, mas falta planejar terminação TLS e encaminhamento confiável do esquema HTTPS. Cookies Secure/CSRF devem funcionar sem afrouxar a segurança.
- O script atual de PostgreSQL cria banco descartável no container de desenvolvimento e pressupõe diretórios e contexto Docker locais. Será migrado para um laboratório próprio.
- Validação Liquibase usa SQL versionado e scripts de ciclo completo. Sua orquestração não unitária deve integrar o novo projeto; changelogs e SQL de produção permanecem exclusivamente em `expenses-liquibase`.

## 3. Organização proposta

```text
Finances/
  expenses-service/
    Expenses.UnitTests/          # xUnit: regras e unidades isoladas
  expenses-ui/
    src/**/*.test.ts(x)          # funções, hooks e componentes isolados
  expenses-tests/
    README.md
    integration/                # .NET/xUnit: HTTP em processo, banco, migrations
    api/                        # Playwright API: HTTP contra URL publicada
    e2e/                        # Playwright: navegador, tela e jornadas
    performance/                # k6: baseline, carga, pico, duração prolongada
    security/                   # ZAP, cenários de abuso e configuração dos scanners
    external/                   # xUnit: Cognito real, execução explícita
    support/                    # dados, clientes e composição exclusivos de teste
    environment/                # Compose isolado, TLS, provedor de teste, observabilidade
    scripts/                    # preparar, executar, coletar evidência, limpar
    reports/                    # resultados gerados; ignorados no Git
```

`expenses-tests` é um projeto de qualidade com runners adequados a cada tarefa, não um único csproj contendo todas as linguagens. A solução .NET do novo projeto pode referenciar os projetos de aplicação para testes de integração. Os testes externos de API/tela usam URL e contratos, sem importar a implementação. A aplicação nunca referencia o projeto de testes.

No frontend, Vitest + React Testing Library com DOM simulado testa uma unidade isolada, com rede/serviços substituídos. Jornadas que combinam páginas, navegação e backend pertencem ao novo projeto. DOM simulado não comprova política real de cookies, renderização entre navegadores ou HTTPS.

Não duplicar todo teste interno em Playwright. xUnit localiza regras/erros com precisão; Playwright HTTP cobre o contrato pela rede; Playwright de navegador cobre comportamento que depende do navegador e das telas.

Versões das ferramentas devem ser fixadas e verificadas contra .NET 10/Node 24/Vite do projeto durante a implementação; não instalar versões flutuantes automaticamente em cada execução.

## 4. Laboratório local e identidade

### Perfil padrão: completamente local

Fluxo proposto: runner → proxy HTTPS local → API em Kestrel → PostgreSQL isolado. Frontend publicado serve na mesma origem, com chamadas `/api/`. Um perfil adicional inicia duas instâncias da API compartilhando banco, chave HMAC e key ring de teste.

O laboratório terá projeto Compose próprio, rede, portas e volumes separados, identificados por execução. Não usar nomes fixos que colidam com o desenvolvimento. A limpeza deve operar somente nos recursos desse projeto, nunca executar exclusão ampla de volumes Docker. Liquibase aplica o master antes da prontidão funcional; `/health/db` isoladamente não comprova schema atualizado.

HTTPS usa autoridade/certificado local confiável para os runners e navegadores. O proxy encaminha Host e esquema; a API confia somente no proxy do laboratório. Não desabilitar Secure/CSRF nem ignorar erros TLS globalmente para fazer o teste passar. Ensaiar também URL com `/api/`, cookies grandes/fragmentados e SameSite no navegador real.

Como Cognito é um serviço externo, a suíte offline precisa de um provedor controlado **exclusivo do projeto de testes**. Proposta: composição de host de teste que reutiliza pipeline/casos de uso reais e substitui a porta IIdentityProvider por um adaptador determinístico. Ele deve simular credenciais, erros, atrasos e rotação e emitir JWT RS256 verificáveis com chaves de teste. O validador criptográfico real permanece ativo, com issuer/client e metadados de teste explícitos; nenhuma assinatura é simplesmente aceita.

Essa composição será um executável/imagem de teste separado, ausente da imagem de produção. Não criar flag de fallback na API publicada; indisponibilidade do Cognito nunca habilita autenticação simulada. Em duas instâncias, o estado de identidade de teste deve ser compartilhado/controlado para que o cenário não dependa de qual instância recebeu a chamada.

O adaptador local não é uma implementação completa do Cognito. Testes de contrato do SDK permanecem isolados; o laboratório offline comprova regras e integração local, não compatibilidade com a AWS. Não depender de um emulador que prometa suporte completo sem verificar as operações utilizadas, especialmente rotação de refresh.

### Perfil externo preservado, desativado por padrão

Mover os três testes Cognito reais para `external`. Não executá-los como parte do comando local padrão. No futuro, podem rodar com aplicação local + Cognito de teste, ou na nuvem. Cada relatório identifica claramente `identity=simulated` ou `identity=cognito`.

Não provisionar AWS nem usar credenciais reais para carga/ataques nesta primeira fase. Valores de 15 minutos e sete dias continuam vigentes; relógio controlado só existe no host/testes apropriados. Ensaios com prazos acelerados serão identificados e não substituirão verificações com prazos reais.

## 5. Matriz funcional, API e navegador

Cada cenário terá ID estável, requisito de origem, pré-condição, ação, resultado e camada. Um cenário marcado como não implementado não poderá aparecer como aprovado.

| Área | Casos prioritários | Evidência |
| --- | --- | --- |
| Cadastro | Entrada válida/inválida, senha, duplicidade, replay, concorrência, commit ambíguo, compensação | Status/corpo; conta/perfil/diário consistentes; nenhuma credencial modificada indevidamente |
| Login | Sucesso, credencial inválida, indisponibilidade, perfil ausente | Cookies somente em sucesso; erro genérico; indisponibilidade não incrementa falhas |
| Bloqueio | 7ª/8ª/9ª/10ª falhas, normalização de e-mail, expiração, reset, duas instâncias | Avisos 3/2/1; bloqueio de 15 min; nenhum bloqueio por IP; limites não contornados por concorrência |
| Sessão | Válida, access expirado/ausente, refresh inválido, prazo absoluto | Respostas/cookies corretos; sete dias não prorrogados |
| Revogação | Cookies copiados, outra instância, logout repetido, logout versus refresh | Novas autorizações negadas após commit; atualização tardia não reativa sessão |
| Falhas | Banco/provedor indisponível, processo interrompido, timeout/cancelamento | Sem sucesso falso ou fallback; estado incerto identificável; limpeza de cookies adequada |
| HTTP/TLS | Content-Type, 8 KB também sem Content-Length, headers, origem, CSRF, prefixo /api | Mesmo contrato no host publicado; certificados e cookies efetivamente validados |
| JWT/chaves | Assinatura/tipo/issuer/client errados, expiração, rotação, reinício do key ring | Rejeição segura; comportamento de rotação e recuperação explicitamente documentado |
| Tela, quando disponível | Preencher/clicar, erro, foco, teclado, loading, clique duplo, recarga, lembrar somente e-mail | Comportamento visível e requisições corretas, sem guardar senha/tokens em storage |
| Navegador, quando disponível | Múltiplas abas, renovação/logout, histórico, Chromium/Firefox/WebKit | Sessão e dados privados coerentes; nenhuma restauração indevida após logout |

Playwright API não abre uma página. Playwright E2E abre a aplicação e interage por elementos visíveis. Compartilhar helpers de autenticação quando adequado não deve fazer um teste de tela pular o login visual que ele pretende comprovar.

A suíte HTTP terá checks de status, schema, headers, cookies e efeitos relevantes. Produzir requisição sem conferir resultado não é um teste aprovado. Fixtures administrativas podem preparar/consultar banco do laboratório; os passos do cenário de caixa-preta não devem contornar a API para fazê-lo passar.

## 6. Performance e carga local

### Objetivos e limites

Usar build Release, limites de CPU/memória registrados, mesmo conjunto de dados e máquina/Docker identificados. Não comparar execução em Debug com Release. Não rodar ZAP, builds ou outras suítes simultaneamente ao ensaio de performance.

Containers separados no mesmo computador continuam disputando CPU, memória e I/O. Coletar métricas do gerador e da aplicação; taxa não entregue pelo gerador invalida uma conclusão de capacidade. Resultados representam o laboratório, não a capacidade AWS ou a experiência de rede dos usuários.

Base inicial: 1.000 identidades/perfis sintéticos consistentes no provedor local e no banco. Preparação ocorre fora da janela medida e é registrada. Cadastros novos são medidos em cenário próprio. Não inventar vínculo Cognito para declarar integração real.

Hipóteses anteriores de 50 ativos/100 no pico continuam exploratórias. O usuário confirmou somente 1.000 cadastrados. Não apresentar 50/100 como demanda de negócio validada.

### Ensaios propostos

| Ensaio | Configuração inicial sugerida | Conclusão permitida |
| --- | --- | --- |
| Sanidade | 1 usuário, 2–5 min | Script, autenticação e coleta funcionam |
| Baseline por operação | Aquecimento identificado; amostras suficientes por operação; repetir 3 vezes | Distribuição local de latência e variação entre execuções |
| Carga gradual de consultas | 1 → 5 → 10 req/s, aproximadamente 5 min por patamar | Curva de latência/recursos da consulta autenticada |
| Jornadas de usuários | 10 → 25 → 50 ativos, pausas realistas; cenário separado de taxa fixa | Funcionamento concorrente sem login artificial a cada consulta |
| Pico | Até 100 ativos ou duplicação de taxa validada, em ensaios separados e limitados | Degradação/recuperação local, sem afirmar equivalência entre VUs e RPS |
| Duração prolongada | 1 hora na carga estável comprovada | Renovações e tendência de memória/conexões |
| Estresse | Aumentos pequenos até limite de erro/recurso predefinido | Ponto de saturação local e recuperação, sem buscar travar o computador |

Começar a baseline com pelo menos 1.000 observações por operação quando viável no provedor local; ampliar quando a variabilidade exigir. A amostra não transforma p99 em garantia estatística. Não acumular milhares de consultas para esconder poucas amostras de login/refresh/logout.

Login/cadastro/refresh/logout têm cenários próprios e mistura explícita nas jornadas. Cookies pertencem a cada usuário virtual; contas compartilhadas só nos cenários deliberados de disputa. Validar o fluxo CSRF. Não emitir JWT diretamente no script e pular login para afirmar que o ciclo completo foi testado.

Medir por operação: p50/p95/p99, quantidade, duração, taxa pretendida/atingida, erros inesperados, timeouts e iterações não iniciadas. k6 exige configuração explícita dos limites de aprovação; checks de resposta sozinhos não asseguram que o processo falhe. Medir tempo completo percebido pelo cliente separadamente de duração interna e handshake/conexão; métricas HTTP nativas não incluem necessariamente todos esses componentes.

Separar inicialização fria e execução aquecida, sem ocultar resultados. `401`/`409`/`423` deliberados devem ser validados e classificados como resultado esperado apenas em seu cenário, nunca globalmente excluídos da contagem.

p95 < 150 ms será reportado por operação. No perfil offline, um resultado abaixo de 150 ms é apenas baseline interna. Os tempos Cognito reais já medidos continuam válidos como evidência exploratória externa e não são substituídos por um provedor instantâneo. Margens de regressão relativas serão definidas depois de medir ruído de máquina, sem relaxar silenciosamente a meta absoluta.

## 7. Segurança local

Quatro frentes complementares:

1. **Unidades e cenários de segurança:** JWT, cookies, CSRF, revogação, concorrência, falha fechada, tentativas distribuídas, bloqueio malicioso de conta, corpo inválido/grande e headers adulterados.
2. **Código e cadeia de dependências:** auditoria NuGet/npm, detecção de segredos com Gitleaks em modo redigido, analisadores de código e revisão dirigida de C#/TypeScript. Incluir imagem de container/configuração antes da etapa de publicação. Escolher regras e versões verificadas; sem correções automáticas que atualizem dependências incompatíveis.
3. **Aplicação executando:** ZAP passivo primeiro; depois varredura ativa limitada ao host de teste. Importar o OpenAPI gerado do mesmo build como artefato privado — não publicar Swagger ou listagem de usuários em produção para facilitar o scanner. Configurar cookie/CSRF e confirmar que o scanner alcança páginas/rotas protegidas, não somente erros de autenticação.
4. **Revisão de requisitos:** mapear os controles aplicáveis do OWASP ASVS nível 2, registrando satisfeitos, pendentes e não aplicáveis. Ausência de MFA, política de senha, enumeração por cadastro e e-mail não verificado precisam de avaliação explícita; o scanner não aprova decisões de produto.

Testes de SameSite/CSRF precisam também de navegador real e origem de ataque local distinta: um cliente HTTP comum pode enviar headers/cookies que o navegador não enviaria. Revalidar essa evidência com telas quando disponíveis.

ZAP ativo terá lista de destinos permitidos e limite de duração; não apontar para ambiente de desenvolvimento, sites externos ou endpoints AWS. Relatórios/traces/HAR podem carregar credenciais: suprimir/redigir headers e dados sensíveis, restringir acesso e retenção e não versionar artefatos brutos. Evidência limpa é parte do aceite.

Qualquer acesso indevido confirmado é bloqueador. Achados altos/críticos confirmados exigem correção ou decisão explícita documentada antes de publicação; falso positivo tem justificativa, não simples exclusão. Uma suíte verde não é certificação nem substitui revisão independente futura.

## 8. Execução, evidências e aceites

Interface planejada de execução, ainda não existente: unit backend, unit frontend, integration, api-local, e2e-local, performance-local, security-local e external-cognito. Um comando de orquestração local prepara o laboratório, verifica prontidão, executa o grupo selecionado, coleta resultados e limpa apenas seus próprios recursos mesmo após falha. Comando externo nunca roda por padrão.

Não renomear/apagar o comando existente antes de atualizar README/CI. Pode haver encaminhamento temporário documentado; nenhum teste não unitário deve continuar escondido sob o comando unitário do backend.

| Momento | Verificação proposta |
| --- | --- |
| Durante desenvolvimento | Unitários do projeto alterado |
| Antes de integrar mudança | Build/lint, unitários, integrações pertinentes, migrations, auditoria de dependências/segredos |
| Após alterar contrato/pipeline/infra local | HTTP pela rede e navegador aplicável |
| Após alterar autenticação/segurança | Matriz de segurança, recuperação, concorrência e ZAP local aplicável |
| Ao estabelecer baseline ou alterar desempenho | k6 com ambiente controlado e comparação da mesma topologia |
| Futuro candidato de release | Execução integrada na nuvem com Cognito e topologia de destino |

Relatório por execução: ID, commits ou identificação das alterações locais dos projetos, versões/digests, tipo de identidade, limites do ambiente, dataset, data, aprovados/falhos/ignorados/não implementados, tempos/amostras, erros, recursos e limpeza. Um relatório agregado aponta a evidência de cada runner; a contagem total não substitui a matriz de requisitos. Falhas intermitentes não viram sucesso por repetir até passar; preservar primeira falha e resultado da repetição.

Aceite da reorganização: mapear os 98 casos e preservar os 46 isolados + 49 integrações locais + 3 externos. Os três externos ficarão marcados como não executados no ciclo local, sem anunciar 98 aprovações. Manter os cenários equivalentes quando uma classe precisar ser dividida.

Aceite do laboratório: dados de desenvolvimento intactos; HTTPS/cookies válidos; conexão PostgreSQL real; duas instâncias sem bypass; simulação explicitamente identificada; falhas sem autenticação indevida; teardown testado.

Aceite de performance: baseline repetível, volume realmente entregue e resultados por operação. A taxa máxima de erros inesperados e carga de lançamento serão aprovadas após baseline; proposta inicial de avaliação é abaixo de 0,1% em carga nominal, sem tolerância a violação de segurança/consistência. Não é aceite já acordado.

## 9. Sequência de implementação

| Etapa | Entrega | Critério de conclusão |
| --- | --- | --- |
| T0 | Manifesto dos casos e estrutura de expenses-tests | Correspondência dos 98 casos, dependências e comandos documentada |
| T1 | Separar unitários; migrar testes HTTP/PG/externos e validação do schema | Backend sem dependência de WebApplicationFactory/banco nos testes unitários; integração preservada; nuvem opt-in |
| T2 | Laboratório local isolado, TLS e identidade controlada | Sobe/limpa sem afetar desenvolvimento; mesmo pipeline aplicativo exercitado; ausência de fallback na imagem de produção |
| T3 | Testes HTTP contra Kestrel/proxy e duas instâncias | Sucesso/falha/concorrência/revogação comprovados pela rede |
| T4 | Unitários frontend e jornadas Playwright | Unitários junto dos componentes implementados; jornadas reais dependem das telas de identidade E5 |
| T5 | Observabilidade mínima, k6 e recuperação | Baseline reproduzível, carga, pico/duração e diagnóstico de falha |
| T6 | Verificações de segurança e relatório consolidado | Escopo autenticado comprovado, achados triados, evidências sem segredos |
| T7 | Futuro: transportar o laboratório para nuvem | Perfis/configuração reaproveitados; Cognito real e infraestrutura final revalidados |

Segurança é transversal desde T1; T6 consolida a varredura e as evidências. T4 não precisa bloquear testes HTTP/carga enquanto não houver telas. Não desenvolver telas ou infraestrutura AWS implicitamente como parte deste levantamento.

## 10. Referências

- [Entrega atual E3/E4 e evidências](../../fdd/FDD-Criacao-Usuario-Autenticacao/2-desenvolvimento.md#login-e-sessoes-e3-e4).
- [Guidelines do projeto](../../guidelines/README.md).
- [Vitest — ambientes e DOM simulado](https://vitest.dev/guide/environment.html).
- [React Testing Library — testes de componentes](https://testing-library.com/docs/react-testing-library/intro/).
- [Playwright — HTTP e contexto de cookies](https://playwright.dev/docs/api/class-apirequestcontext).
- [Playwright — isolamento de navegador](https://playwright.dev/docs/browser-contexts).
- [k6 — critérios automáticos de aprovação](https://grafana.com/docs/k6/latest/using-k6/thresholds/).
- [k6 — automação de performance](https://grafana.com/docs/k6/latest/testing-guides/automated-performance-testing/).
- [ZAP — automação](https://www.zaproxy.org/docs/automate/automation-framework/).
- [ZAP — API Scan ativo e modo passivo](https://www.zaproxy.org/docs/docker/api-scan/).
- [OWASP ASVS](https://owasp.org/projects/asvs).
- [NuGet — dependências vulneráveis, inclusive transitivas](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list).
- [npm audit](https://docs.npmjs.com/cli/npm-audit/).
- [Gitleaks](https://github.com/gitleaks/gitleaks/blob/master/README.md).
