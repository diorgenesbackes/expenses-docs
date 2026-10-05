### HLD: Contas da Casa MVP Finances

Versão: 1.1
Data: 2026-10-01
Responsável: Diórgenes Backes

---

### Objetivo técnico
Construir um sistema web para centralizar as finanças de uma casa com segurança,
persistência transacional, isolamento por casa e histórico imutável dos
fechamentos mensais.

A primeira versão utilizará um backend único em C# e ASP.NET Core (.NET),
organizado como monólito modular. Cadastro, autenticação, casas, finanças e
renda extra serão implementados na mesma aplicação, com um único deploy.
Todas as operações da aplicação serão expostas por endpoints REST e concluídas
no ciclo de requisição e resposta. Não haverá funções Lambda, mensageria,
processamento em segundo plano ou comunicação em tempo real nesta versão.

Dependências com outros sistemas
- Amazon Cognito User Pools para credenciais, autenticação e emissão de tokens,
  acessado pelo backend.
- Serviços gerenciados da AWS para hospedagem, banco, observabilidade,
  criptografia, proteção de borda e backup.
- Não haverá integração com bancos, corretoras ou plataformas de investimento.

Esta revisão substitui a proposta anterior de microsserviços híbridos. Descreve
a arquitetura a implementar; a POC local em JavaScript e Python continua sendo
uma implementação anterior, separada do backend planejado.

---

### Arquitetura geral
O frontend React com TypeScript será hospedado no AWS Amplify Hosting e
consumirá uma API REST JSON versionada em `/v1`. O backend ASP.NET Core será
empacotado em Docker e executado no ECS Fargate, atrás de um Application Load
Balancer (ALB). O Amazon API Gateway será a entrada pública HTTPS das APIs
REST `/v1`, responsável pela gestão das rotas, limites de requisição e políticas
de acesso da API. Sua integração privada, por VPC Link, encaminhará chamadas
ao ALB e ao backend. O ALB não será exposto como entrada pública alternativa.
A aplicação fará autenticação de sessão, autorização por casa e papel,
validação, execução das regras e acesso ao banco.

O backend terá quatro módulos internos: Identity, Household, Finance e Income.
Eles se comunicarão por chamadas de métodos no mesmo processo, sem chamadas
HTTP internas ou eventos. A divisão preserva a organização por domínio, mas
não implica deploys, bancos ou infraestrutura independentes.

O banco será Amazon Aurora PostgreSQL Serverless v2. O acesso será realizado
com Entity Framework Core e o provider Npgsql, com schemas `identity`,
`household`, `finance` e `income` para organização. Cada módulo concentra as
escritas de seu domínio; a aplicação possui uma estratégia única de migrações
e pode usar transações locais abrangendo mais de um módulo quando necessário.

O frontend atualiza sua tela a partir da resposta de uma escrita e de consultas
GET ao abrir, atualizar ou voltar a uma tela. Alterações de outro usuário serão
visíveis na próxima consulta REST. Não há garantia de propagação em `500 ms`,
assinaturas WebSocket ou sincronização automática em tempo real no MVP.

Ambiente de implantação
- Cloud AWS, com ambientes separados para `dev`, `hlg` e produção.
- Frontend no Amplify Hosting.
- API Gateway como entrada pública REST, integrado por VPC Link ao ALB interno.
- Um único backend ASP.NET Core em ECS Fargate atrás de ALB.
- Aurora PostgreSQL Serverless v2 para persistência.
- Desenvolvimento local com backend .NET e PostgreSQL local.

Tecnologias principais
- React e TypeScript.
- C#, .NET e ASP.NET Core Web API, em versão LTS suportada definida na implementação.
- Entity Framework Core e Npgsql.
- Amazon API Gateway, VPC Link, Application Load Balancer, Docker e ECS Fargate.
- Aurora PostgreSQL Serverless v2 e Amazon Cognito User Pools.
- OpenAPI para documentação dos contratos REST.

Padrões adotados
- Monólito modular com um único artefato de backend e deploy.
- Separação entre endpoints, casos de uso, regras de domínio e infraestrutura.
- Comunicação síncrona via REST externamente e métodos internamente.
- Transações locais para operações financeiras e alterações relacionadas.
- Valores monetários em `decimal` no .NET e `numeric` no PostgreSQL, com
  precisão e arredondamento explicitados nos contratos.
- Concorrência otimista com versão do período e bloqueio temporário de edição.
- Snapshots imutáveis de fechamento e escritas idempotentes quando aplicável.

O uso de `async`/`await` para I/O no .NET é um detalhe de implementação: a
requisição permanece aberta até a conclusão da operação. Não representa
processamento assíncrono de negócio, filas ou respostas de aceite para jobs.

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependências |
| ----------- | ----------------- | ------------ |
| Frontend Web | Consumir REST, exibir indicadores, versões, conflitos e resultados de operações. | Amplify Hosting, backend .NET |
| Backend ASP.NET Core | Implementar `/v1`, autenticar sessões, autorizar acesso, executar casos de uso e transações. | ALB, Cognito, EF Core, Aurora |
| Amazon API Gateway | Gerenciar a entrada pública REST `/v1`, rotas, throttling, limites e logs de acesso; encaminhar chamadas ao backend. | VPC Link, ALB |
| Módulo Identity | Intermediar cadastro, login, logout e sessão; manter perfil interno e localização por e-mail. | Cognito, schema `identity` |
| Módulo Household | Gerenciar casas, membros, papéis, isolamento e exclusão com recuperação temporária. | Identity, schema `household` |
| Módulo Finance | Gerenciar períodos, despesas, pagamentos, salários, benefícios, rateio, fechamentos, bloqueios, migração e exportação. | Household, schema `finance` |
| Módulo Income | Gerenciar fontes recorrentes, aluguéis, renda variável e consolidação de renda extra. | Household, schema `income` |
| Amazon Cognito User Pools | Manter credenciais e emitir tokens. | Módulo Identity |
| Application Load Balancer | Receber a integração privada do Gateway, verificar saúde e distribuir requisições às tasks do backend. | VPC Link, ECS Fargate |
| Aurora PostgreSQL Serverless v2 | Persistir dados com transações e precisão monetária. | Backend via EF Core/Npgsql |
| Secrets Manager e KMS | Gerenciar segredos e criptografia em repouso. | Backend, banco, backups |
| AWS WAF | Proteger a entrada pública da API. | API Gateway REST API |
| CloudWatch Logs e Logs Insights | Centralizar e consultar logs estruturados. | Backend, infraestrutura |
| OpenTelemetry e AWS X-Ray | Rastrear requisições e chamadas ao banco e ao Cognito. | Backend |
| AWS Backup | Apoiar backup e recuperação, junto à recuperação point-in-time do Aurora. | Aurora |

---

### Fluxo de requisições e de dados
**Fluxo de requisição**
- O usuário acessa o frontend no Amplify Hosting.
- Cadastro e login passam pelos endpoints REST do módulo Identity, que acessa
  o Cognito e estabelece a sessão por cookies seguros.
- O frontend envia a requisição HTTPS `/v1` ao API Gateway, que encaminha
  pela integração privada ao ALB e ao backend. Cookies e cabeçalhos necessários
  à sessão são preservados na integração.
- O ASP.NET Core valida a sessão; o caso de uso valida casa, papel e dados.
- O módulo responsável executa a regra e persiste a alteração no Aurora.
- O backend confirma a transação antes de responder com o resultado.
- O frontend atualiza o estado usando a resposta ou uma nova consulta GET.
- Falhas retornam status HTTP e erro estruturado, sem agendar execução posterior.

**Fluxo de dados**
- Cadastro → Identity cria o principal no Cognito e o perfil interno no banco.
  Como Cognito e PostgreSQL não compartilham transação, o caso de uso deve
  tratar retries idempotentes e compensação de falhas parciais.
- Vínculo de membro → Household consulta Identity por chamada interna, valida
  limite de responsáveis e administrador e persiste o vínculo.
- Edição mensal → Finance valida permissão, estado, versão e bloqueio, salva
  a alteração e incrementa a versão na mesma transação. Versão desatualizada
  resulta em `409 Conflict`; o frontend consulta o estado e apresenta o conflito.
- Bloqueio temporário → Finance registra expiração de `5 minutos`; atividade
  renova o bloqueio por REST. Cada operação verifica a expiração no banco;
  bloqueios expirados deixam de valer sem tarefa em segundo plano.
- Fechamento → Finance valida pagamentos, permissões, concorrência e valores,
  calcula rateio e grava `MonthlyPeriodVersion` e `AllocationSnapshot` na mesma
  transação. A resposta confirma o fechamento concluído.
- Reabertura → usuário autorizado reabre por REST; um novo fechamento cria
  outra versão, preservando snapshots anteriores.
- Renda extra → Income salva fontes e lançamentos e retorna os dados atualizados.
- Migração e exportação → endpoints executam a operação dentro da requisição,
  com limites de volume e tempo. Exportação entrega o arquivo na resposta;
  migração retorna o resultado concluído. Volumes maiores exigem divisão em
  lotes REST explícitos, sem filas ou jobs nesta versão.
- Exclusão da casa → Household registra exclusão lógica e prazo de recuperação
  de `30 dias`, bloqueando acesso comum imediatamente. A eliminação definitiva
  é executada por endpoint administrativo REST, restrito à operação, após o
  prazo; o backend verifica a elegibilidade e conclui a exclusão na requisição.
  A rotina operacional deve assegurar a execução, sem scheduler na aplicação.

---

### Modelo de dados (alto nível)
Entidades principais
- `User`: perfil interno vinculado ao usuário autenticável do Cognito.
- `Household`: casa e estado do ciclo de vida, incluindo exclusão lógica.
- `HouseholdMember`: vínculo do usuário com a casa e papel
  `household_admin`, `financial_member` ou `viewer`.
- `MonthlyPeriod`: estado operacional atual do mês, versão vigente, status e
  bloqueio temporário.
- `MonthlyPeriodVersion`: snapshot imutável criado a cada fechamento.
- `ExpenseCategory`: categoria de despesa, recorrência e estado ativo.
- `ExpenseEntry`: lançamento monetário por categoria e período.
- `PaymentStatus`: estado pago ou pendente da categoria no período.
- `SalaryEntry`: salário líquido mensal por responsável financeiro.
- `Benefit`: benefício compartilhado e prioridade de abatimento.
- `BenefitEntry`: valor mensal do benefício.
- `AllocationSnapshot`: resultado do rateio utilizado em um fechamento.
- `IncomeSource`: fonte recorrente de renda extra.
- `IncomeEntry`: lançamento mensal recorrente ou variável.
- `AuditLog`: registro de alterações sensíveis.
- `MigrationRecord`: registro do resultado de uma migração histórica síncrona.
- `ExportRecord`: registro de uma exportação síncrona.

Relações
- Um `User` pode participar de múltiplos `Household`.
- Um `Household` possui múltiplos `HouseholdMember` e ao menos um
  `household_admin`.
- Um `Household` possui no máximo dois responsáveis financeiros no MVP e pode
  possuir usuários adicionais com papel `viewer`.
- Um `Household` possui múltiplos `MonthlyPeriod`.
- Um `MonthlyPeriod` possui múltiplas versões imutáveis em
  `MonthlyPeriodVersion`.
- Um `MonthlyPeriod` agrega lançamentos, pagamentos, salários, benefícios e
  resultados de rateio.
- Uma `ExpenseCategory` possui múltiplos `ExpenseEntry`.
- Um `Benefit` possui múltiplos `BenefitEntry`.
- Um `MonthlyPeriodVersion` referencia um `AllocationSnapshot`.
- Uma `IncomeSource` possui múltiplos `IncomeEntry`.

Fonte de verdade
- Aurora PostgreSQL é a fonte de verdade dos dados de domínio.
- Cognito é a fonte de verdade das credenciais e principais autenticáveis.
- O backend único controla todas as escritas, organizadas por módulo.
- O frontend não acessa o banco diretamente e não mantém persistência financeira
  autoritativa no navegador.
- Não haverá cache distribuído no MVP.

---

### Interfaces públicas
| Nome | Tipo | Protocolo | Exposição | Metas/Limites |
| ---- | ---- | ---------- | --------- | ------------ |
| `/v1/identity` | API | REST JSON sobre HTTPS | Usuários | `p95 < 150 ms` para operações frequentes, a validar |
| `/v1/households` | API | REST JSON sobre HTTPS | Usuários | `p95 < 150 ms` para operações frequentes |
| `/v1/finances` | API | REST JSON sobre HTTPS | Usuários | `p95 < 150 ms` para operações frequentes |
| `/v1/incomes` | API | REST JSON sobre HTTPS | Usuários | `p95 < 150 ms` para operações frequentes |
| `/v1/admin` | API | REST sobre HTTPS | Operação restrita | Operações administrativas síncronas |

Todos os grupos de rotas pertencem ao mesmo backend. Migrações e exportações
terão limites específicos de tamanho e duração, a definir. Não haverá endpoints
de jobs assíncronos, streams, filas ou canais de eventos.

---

### Considerações de escalabilidade e disponibilidade
- O backend é sem estado de sessão em memória e pode escalar em réplicas no ECS.
- API Gateway aplica throttling e limites de requisição na entrada pública.
- ALB distribui chamadas e usa health checks de liveness e readiness.
- Versões e bloqueios ficam no banco para funcionar entre réplicas.
- A escala abrange o backend inteiro; módulos não escalam independentemente.
- Produção deve dimensionar réplicas e redundância conforme a meta de disponibilidade.
- Pools de conexão e capacidade do Aurora devem acompanhar a escala do backend.
- Backups automatizados e recuperação point-in-time devem ser validados com
  testes de restauração.

Metas, sujeitas à validação de infraestrutura e carga
- Disponibilidade mensal mínima de `99,9%` em produção.
- Latência `p95 < 150 ms` para operações frequentes.
- `RPO` de até `5 minutos` e `RTO` de até `60 minutos`.
- Atualização de dados compartilhados na próxima consulta REST, sem SLO de
  entrega em tempo real.

---

### Segurança
Autenticação
- Identity intermedeia chamadas ao Cognito para cadastro, login, logout e sessão.
- Tokens permanecem em cookies `HttpOnly`, `Secure` e `SameSite`, conforme o
  contrato de sessão a detalhar. Não são expostos ao JavaScript do frontend.
- A aplicação valida tokens e sessão no ASP.NET Core. O Gateway não substitui
  a autorização por casa e papel realizada pelo backend.
- Definir proteção contra CSRF, política CORS com origens explícitas e envio
  de credenciais considerando os domínios do frontend e da API.
- Cada usuário cria sua conta; membros são localizados por e-mail.
- Recuperação de senha permanece fora do MVP conforme o FDD de autenticação.

Autorização
- Toda leitura e escrita valida o usuário autenticado e seu vínculo com a casa.
- Papéis por casa são consultados no banco e não presumidos a partir do token.
- `household_admin` administra casa, membros, períodos, migrações e exportações.
- `financial_member` registra dados financeiros e consulta indicadores.
- `viewer` consulta dados.
- Uma casa mantém ao menos um administrador e no máximo dois responsáveis
  financeiros no MVP.
- Endpoints administrativos de eliminação definitiva têm autorização operacional
  própria, sem acesso para membros comuns da casa.

Proteção de dados
- Criptografia em trânsito e em repouso com KMS.
- Logs não registram senhas, tokens ou valores financeiros sensíveis.
- Exclusão lógica bloqueia acesso comum durante os `30 dias` de recuperação.
- Após o prazo, a operação deve executar a eliminação de dados financeiros e PII
  do banco ativo, ressalvadas retenções legalmente aplicáveis.
- Backups expiram conforme retenção operacional; uma restauração deve reaplicar
  exclusões antes de disponibilizar os dados aos usuários.
- Logs retidos usam identificadores pseudonimizados.
- Política de retenção e exclusão depende de validação jurídica.

Gestão de segredos
- Secrets Manager armazena credenciais operacionais; KMS gerencia as chaves.
- IAM aplica privilégio mínimo à task do backend.
- WAF protege o API Gateway; throttling no Gateway e rate limiting no backend
  restringem abuso e tentativas de login.

---

### Observabilidade
- Logs estruturados no CloudWatch, consultáveis por Logs Insights, com retenção
  por ambiente e mascaramento de dados sensíveis.
- Registrar correlation ID, trace ID, módulo, endpoint, resultado e código de erro.
- Medir latência `p50`, `p95` e `p99`, erros `4xx`/`5xx`, disponibilidade,
  falhas de autenticação, conflitos de concorrência e chamadas ao Cognito.
- Monitorar erros, throttling e latência de integração do API Gateway,
  preservando correlation IDs entre Gateway, ALB e backend.
- Monitorar CPU, memória, tasks, conexões e capacidade do banco, falhas de
  backup e restauração e casas com eliminação definitiva pendente.
- OpenTelemetry instrumenta requisições HTTP e dependências do backend, com
  exportação para X-Ray e amostragem definida na implementação.
- Dashboards e alertas cobrem SLOs, API Gateway, backend, ALB, Aurora e exclusões pendentes.

---

### Riscos arquiteturais e mitigação
#### Vazamento de dados entre casas
- Validar vínculo e papel em todo caso de uso, inclusive exportação e migração.
- Testar isolamento e registrar acessos negados.
- Em incidente, bloquear a operação afetada e investigar acessos.

#### Sobrescrita de edições concorrentes
- Validar versão e bloqueio dentro da transação, com atualização condicional.
- Retornar conflito e permitir recarga do estado antes de nova edição.
- Preservar snapshots imutáveis dos fechamentos.

#### Visualização temporária de dados desatualizados
- Sem atualização em tempo real, outro usuário pode alterar dados já exibidos.
- Consultar REST ao abrir ou retomar a tela e oferecer atualização explícita.
- Validar versão em toda escrita para impedir sobrescrita silenciosa.

#### Acoplamento entre módulos e impacto de falhas
- Manter contratos internos claros e regras concentradas no módulo responsável.
- Um deploy ou falha pode afetar toda a API; usar health checks, rollback e
  testes de integração dos fluxos principais.
- Avaliar extração de serviços somente quando houver necessidade comprovada.

#### Operações síncronas excederem limites de requisição
- Limitar volume de importações e exportações e definir timeouts e tamanhos
  de payload coerentes com API Gateway, ALB e backend.
- Dividir migrações maiores em lotes explícitos e idempotentes.
- Não retornar sucesso antes da conclusão nem migrar silenciosamente para jobs.

#### Eliminação definitiva depender de execução operacional
- O prazo expira independentemente da execução do endpoint administrativo.
- Definir responsável e frequência da rotina e alertar sobre exclusões pendentes.
- Validar capacidade, limites de volume e cumprimento da política de retenção.

---

### ADRs e próximos passos
ADRs a elaborar conforme esta revisão
- `ADR-001`: backend único .NET como monólito modular, em Docker/ECS Fargate.
- `ADR-002`: PostgreSQL com EF Core/Npgsql e schemas por módulo.
- `ADR-003`: API Gateway para gestão REST, integração privada com ALB e
  endpoints síncronos; chamadas internas no mesmo processo.
- `ADR-004`: atualização do frontend por consulta REST na primeira versão.
- `ADR-005`: Cognito intermediado pelo backend e autorização por casa e papel.
- `ADR-006`: versionamento, snapshots e bloqueios com expiração verificada em REST.
- `ADR-007`: frontend React no Amplify Hosting.
- `ADR-008`: observabilidade do backend com CloudWatch e OpenTelemetry/X-Ray.

Decisões pendentes
- Definir versão LTS do .NET, estrutura da solução e limites dos módulos.
- Detalhar contratos REST, OpenAPI, erros, idempotência e concorrência.
- Definir cookies, renovação de sessão, CSRF, CORS e domínios de implantação.
- Evoluir modelo relacional, índices e precisão monetária por migrações Liquibase
  em YAML/SQL no projeto `expenses-liquibase`; EF Core/Npgsql permanece responsável
  pelo acesso aos dados. Ver [modelo físico da sprint-1](db/DB-Modelo-Fisico.md).
- Definir rotas, domínio público, throttling e integração VPC Link do Gateway.
- Definir limites de importação/exportação, payloads e timeouts compatíveis
  entre API Gateway, ALB e aplicação.
- Definir IaC, CI/CD, rede, IAM, dimensionamento e retenção por ambiente.
- Definir procedimento operacional para eliminação definitiva via REST.
- Validar política de retenção e exclusão juridicamente.

Próximos passos
- Elaborar os ADRs e os LLDs dos módulos do backend .NET.
- Alinhar PRD, FDD de autenticação e diagramas C4 com esta revisão: referências
  anteriores a microsserviços, Lambda e atualização em tempo real
  nesses artefatos não representam a decisão atual do HLD.
- Criar a solução ASP.NET Core, contratos OpenAPI e migrações iniciais.
- Implementar fluxos REST de identidade, casas, finanças e renda extra.
- Validar isolamento, concorrência, cálculos e consistência dos fechamentos.
- Validar latência, limites síncronos e restauração de backups.
- Definir IaC, pipeline e ambientes `dev`, `hlg` e produção.
