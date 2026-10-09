# Levantamento para modelagem do banco de dados

Data: 2026-10-03  
Status: levantamento de requisitos e proposta preliminar; decisões pendentes identificadas.  
Escopo: backend completo do Contas da Casa / Expenses.

Evolução: a sprint-1 em Liquibase está descrita no [modelo físico](DB-Modelo-Fisico.md), incluindo o tratamento das decisões D01–D24. Este levantamento preserva as propostas e pendências da etapa original.

## 1. Resultado e fontes

A documentação permite iniciar o modelo conceitual dos módulos Identity, Household, Finance e Income. O HLD enumera **17 entidades principais**, mas a implementação relacional também precisa representar prioridades de benefícios, participantes históricos, idempotência e controle compartilhado de tentativas de login. A quantidade final de tabelas depende das decisões registradas neste levantamento.

O ponto central é preservar o que ocorreu em cada mês: alterações atuais em membros, nomes, categorias ou benefícios não podem modificar os dados utilizados em um fechamento anterior.

Este documento reúne requisitos confirmados, estruturas propostas e lacunas. Os nomes de tabelas e campos são sugestões para a próxima etapa. Não foram criadas entidades C#, migrations ou estruturas em um banco.

| Fonte consultada | Uso no levantamento |
| --- | --- |
| [PRD v1.0, 2026-05-30](../PRD.md) | Escopo, RF-01 a RF-20, regras de negócio e critérios de aceite. |
| [HLD v1.1, 2026-10-01](../HLD.md) | Arquitetura vigente, schemas, entidades, transações, concorrência e exclusão. |
| [FDD de criação de usuário e autenticação v1.0](../fdd/FDD-Criacao-Usuario-Autenticacao/FDD-Criacao-Usuario-Autenticacao.md) | Perfil interno, Cognito, idempotência, bloqueio de login e reconciliação. |
| [README da POC](../../extras/README.md) | Comportamento anterior, dados históricos e persistência em Excel. |
| [Análise C4](../../extras/c4/criacao-usuario-autenticacao-c4.md) e diagramas [C1](../../extras/c4/criacao-usuario-autenticacao-c1.puml), [C2](../../extras/c4/criacao-usuario-autenticacao-c2.puml), [C3](../../extras/c4/criacao-usuario-autenticacao-c3.puml), [C4](../../extras/c4/criacao-usuario-autenticacao-c4.puml) | Detalhes complementares da identidade; topologia anterior ao HLD vigente. |
| [Prompt C4](../../extras/06-c4-gernerator.md) e [comando C4](../../extras/07-c4-command.md) | Materiais de geração de documentação; não acrescentam requisitos de negócio. |
| [README do backend](../../expenses-service/README.md) e [README do frontend](../../expenses-ui/README.md) | Estado inicial dos projetos; não definem o modelo de domínio. |
| [Código da POC](../../extras/app.js), especialmente `createInitialState`, `calculateCommitment` e recorrência | Evidência complementar do cálculo anterior e das limitações dos dados de origem. |
| [Exportação da POC](../../extras/server.py), funções `build_cost_rows` e `build_income_rows` | Estrutura dos dados exportados para as abas da POC. |

Também foram conferidos os projetos .NET e o ponto de entrada da API. Atualmente há a estrutura em .NET 10 e o health check, sem configuração de EF Core/Npgsql, entidades de negócio ou migrations nos arquivos examinados. O README do backend ainda descreve somente a API e precisa ser atualizado em uma etapa própria.

A planilha `extras/Contas da Casa.xlsx` e seus backups são fontes potenciais de migração. Este levantamento utiliza sua documentação e o código de exportação; **não realizou auditoria célula a célula dos arquivos Excel**, nem considera que o estado inicial embutido no JavaScript seja a versão mais recente dos dados.

## 2. Base técnica e divergências documentais

Requisitos confirmados no HLD vigente:

- Um backend .NET como monólito modular, com um banco Aurora PostgreSQL Serverless v2 e PostgreSQL local no desenvolvimento.
- EF Core e Npgsql, com estratégia única de migrations.
- Schemas `identity`, `household`, `finance` e `income`; schemas organizam módulos, mas não isolam casas por si só.
- Operações síncronas, concluídas dentro da requisição REST; transações locais podem abranger módulos.
- Valores monetários em `decimal` no .NET e `numeric` no banco.
- Concorrência otimista por versão do período e bloqueio temporário de edição por cinco minutos.
- Fechamentos versionados e imutáveis.
- Exclusão lógica da casa com recuperação por 30 dias; eliminação definitiva por operação administrativa REST.
- Nenhum cache distribuído, fila, job de negócio ou comunicação em tempo real no MVP.

| Divergência | Tratamento neste levantamento |
| --- | --- |
| PRD exige propagação em 500 ms; HLD substitui por consultas REST. | Prevalece o HLD para arquitetura. Não derivar tabelas de eventos de sincronização ou assinaturas. |
| FDD/C4 descrevem Identity Service independente. | Interpretar a responsabilidade como módulo Identity dentro da aplicação única. |
| PRD admite exportação assíncrona; HLD exige conclusão síncrona. | Registros de migração/exportação documentam operações concluídas, sem fila de jobs. |
| PRD situa `viewer` como evolução futura; HLD inclui membros adicionais com esse papel. | Prever `viewer` no modelo e alinhar sua liberação funcional. |
| PRD RF-01 exige envio, aceite, reenvio e cancelamento de convite; HLD descreve associação de usuário localizado por e-mail. | Fluxo de ingresso permanece pendente. O HLD não elimina explicitamente o requisito de convite. |
| FDD exclui convites de sua feature de autenticação. | Isso não exclui convites do produto; eventual convite pertence ao módulo Household. |
| POC usa nomes fixos de dois responsáveis, valores agregados e meses separados por tela. | Tratar como legado. O novo modelo usa usuários/membros e período compartilhado. |

## 3. Cobertura dos requisitos

| Requisito | Informações necessárias à persistência |
| --- | --- |
| RF-01 — Casa e responsáveis | Casa, criador, membros, papéis e, se mantido o fluxo, convite e aceite. |
| RF-02 — Isolamento | Casa em toda entidade pertencente a uma família; vínculo do usuário; integridade das referências entre casas. |
| RF-03 — Períodos | Competência única por casa, estado, versão de edição, aberturas, fechamentos e reaberturas. |
| RF-04 — Categorias | Nome, unicidade por casa, recorrência, ativação/desativação e preservação histórica. |
| RF-05 — Lançamentos | Múltiplos valores por categoria/mês, observação, autoria, edição e remoção rastreável. |
| RF-06 — Somas rápidas | Resultado monetário validado; expressão original opcional, sem execução de fórmulas. |
| RF-07 — Pagamentos | Estado pago/pendente por categoria no período, conforme granularidade explicitada no HLD. |
| RF-08 — Indicadores de custos | Agregação dos lançamentos e pagamentos; definição da janela das médias. |
| RF-09 — Histórico da categoria | Identidade estável da categoria, série mensal e representação de meses sem lançamento como zero na consulta. |
| RF-10 — Salários | Valor líquido por responsável e competência, distinguindo ausência de valor zero. |
| RF-11 — Benefícios | Cadastro, valor mensal, prioridade entre responsáveis e vigência da configuração. |
| RF-12 — Rateio | Entradas completas, percentual inteiro, parcelas, total e diferença. |
| RF-13 — Não negatividade | Benefício aplicado por participante, contribuição em dinheiro e excedente. |
| RF-14 — Histórico do rateio | Snapshot de entradas, configuração e resultados associado à versão do fechamento. |
| RF-15 — Fontes recorrentes | Fonte, nome, estado e valores mensais. |
| RF-16 — Sugestões | Último valor conhecido, origem da sugestão e confirmação/alteração pelo usuário. |
| RF-17 — Rendas variáveis | Múltiplos lançamentos de renda por período, com controle de edição. |
| RF-18 — Indicadores de renda | Consolidação por tipo/fonte/período e histórico. |
| RF-19 — Migração | Validação prévia, confirmação, mapeamento do legado, deduplicação, resultado e autoria. |
| RF-20 — Exportação | Casa, solicitante, recorte, formato, versão do contrato e resultado da geração. |
| FDD de identidade | Perfil/Cognito, unicidade de e-mail, proteção contra tentativas e compensação de falhas parciais. |
| HLD — Concorrência e ciclo de vida | Bloqueio compartilhado, escrita condicional, exclusão, recuperação e eliminação definitiva. |

## 4. Catálogo de entidades e campos candidatos

**Legenda:** “confirmado” significa que o conceito está explicitado na documentação; “proposto” representa uma forma de implementá-lo, não uma decisão já aprovada.

Convenções candidatas: `id` estável nas entidades; `household_id` nas tabelas pertencentes a uma casa; `created_at` e `created_by_user_id` quando houver autoria; `updated_at` e `updated_by_user_id` para dados mutáveis. A autoria é definida pelo backend. Snapshots e registros de auditoria não recebem campos que sugiram edição normal. Nulabilidade de autores em importações históricas precisa refletir informação efetivamente disponível.

### 4.1. Schema `identity`

| Entidade / tabela candidata | Origem | Campos e regras a representar |
| --- | --- | --- |
| `User` / `users` | Confirmado: HLD e FDD | `id`, `cognito_sub`, `email`, `normalized_email`, `created_at`, `updated_at`. `cognito_sub` único; e-mail normalizado único, com regra alinhada à configuração do Cognito. Nome de exibição não é exigido pelo cadastro atual. |
| `LoginAttempt` e/ou `LoginThrottle` | Necessidade confirmada; armazenamento proposto | Chaves derivadas de e-mail/IP, instante das falhas e/ou janela, contador e `blocked_until`. Deve funcionar entre réplicas, com atualização atômica. Escolher janela fixa ou móvel antes de decidir entre contador e eventos de tentativa. |
| `IdentityReconciliation` | Registro persistente proposto; reconciliação exigida no FDD | `id`, identificador da operação, principal Cognito quando conhecido, tipo de divergência, estado, datas e código de falha seguro. Útil se logs não forem suficientes para acompanhamento operacional. Não implica processamento em segundo plano. |

Senhas e tokens não pertencem ao banco de domínio. A sessão é intermediada pelo backend e utiliza cookies/Cognito. Uma tabela local `Session` não é exigida pelas fontes; somente deve surgir se houver novo requisito de sessão/dispositivo ou revogação local.

O FDD usa IDs públicos como `usr_...`. Isso não define o tipo físico da chave: resolver se o identificador público será o próprio ID textual ou uma representação de uma chave interna.

### 4.2. Schema `household`

| Entidade / tabela candidata | Origem | Campos e regras a representar |
| --- | --- | --- |
| `Household` / `households` | Confirmado: RF-01/RF-02 e HLD | `id`, `name`, `created_by_user_id`, datas, `deleted_at`, `deleted_by_user_id`, `recoverable_until`. Moeda e fuso da casa são propostas dependentes de decisão. Não há requisito de nome globalmente único. |
| `HouseholdMember` / `household_members` | Confirmado: RF-01/RF-02 e HLD | `id`, `household_id`, `user_id`, `role`, `joined_at`, `left_at` ou estado equivalente. Um vínculo ativo por usuário/casa; ao menos um administrador e até dois responsáveis financeiros. Definir se o papel determina a participação financeira ou se haverá atributo específico. |
| `HouseholdInvitation` / `household_invitations` | Exigido pelo RF-01 se mantido o convite | Casa, e-mail normalizado do destinatário, papel pretendido, autor, hash do token, criação, expiração, aceite, cancelamento e usuário que aceitou. Reenvio e consumo único precisam de regras. |

Ao criar a casa, criar o vínculo administrativo na mesma transação. A alteração/remoção de membros deve preservar referências históricas. Reingresso pode reativar um vínculo ou criar um novo episódio, mas precisa de uma convenção única.

**Papéis documentados:** `household_admin`, `financial_member`, `viewer`. O limite de responsáveis financeiros deve contar todos os participantes do rateio, inclusive administradores que contribuam, conforme a regra a confirmar; contar apenas o nome `financial_member` seria insuficiente.

### 4.3. Schema `finance`: período e despesas

| Entidade / tabela candidata | Origem | Campos e regras a representar |
| --- | --- | --- |
| `MonthlyPeriod` / `monthly_periods` | Confirmado: RF-03 e HLD | `id`, `household_id`, competência, `status`, `edit_version`, autoria/datas de abertura e último fechamento. Competência única por casa. Bloqueio: titular, identificador da aquisição e expiração, na própria linha ou em tabela 1:1. |
| `ExpenseCategory` / `expense_categories` | Confirmado: RF-04 | `id`, `household_id`, `name`, `normalized_name`, `is_active`, `is_recurring`, datas. Nome único na casa segundo normalização definida. Desativar não exclui lançamentos. |
| `PeriodCategory` / `period_categories` | Associação proposta para representar a categoria no mês | `id`, `household_id`, `monthly_period_id`, `expense_category_id`, configuração aplicável naquele mês; opcionalmente nome registrado no período. Única por casa/período/categoria. Pode incorporar o `PaymentStatus` do HLD. |
| `PaymentStatus` | Confirmado no HLD | Estado pago/pendente da categoria no período, quem marcou e quando. Pode ser parte de `period_categories` ou tabela própria com a mesma chave única; escolher uma representação para evitar duplicação. Sem lançamento, não há obrigação de pagamento. |
| `ExpenseEntry` / `expense_entries` | Confirmado: RF-05 | `id`, `household_id`, referência à categoria do período, `amount`, `note`, autoria/datas. Expressão original e origem de importação são opcionais. Um ID por lançamento; não usar categoria/mês como chave única do lançamento. |

A associação `PeriodCategory` é uma proposta para separar configuração atual e aplicação mensal. Alternativa: configuração de categoria com vigência e estado de pagamento separado. Não filtrar histórico pelo `is_active` atual: isso retiraria despesas antigas dos totais.

Remoção de lançamento em rascunho pode ser física com auditoria suficiente ou lógica com filtros explícitos. A política ainda precisa ser definida; aplicar exclusão lógica indiscriminadamente aumenta o risco de somar registros removidos.

### 4.4. Schema `finance`: salários e benefícios

| Entidade / tabela candidata | Origem | Campos e regras a representar |
| --- | --- | --- |
| `PeriodParticipant` / `period_participants` | Proposto para preservar os responsáveis do mês | `id`, casa, período, vínculo do membro e identificação histórica mínima. Único por período/membro. A mudança de membro atual não deve transferir salário ou participação de mês anterior. |
| `SalaryEntry` / `salary_entries` | Confirmado: RF-10 | Casa, período/participante, salário líquido, autoria/datas. Um salário por participante/mês. Linha ausente representa valor não informado; valor zero é permitido e diferente de ausência. |
| `Benefit` / `benefits` | Confirmado: RF-11 | `id`, casa, nome e configuração de utilização; ativação/desativação é proposta. Titular opcional se o produto distinguir o proprietário do benefício da ordem de abatimento. |
| `BenefitPriorityRule` / `benefit_priority_rules` | Forma relacional proposta para RF-11 | Benefício, versão/vigência da regra, participante/membro e posição de abatimento. Definir também a ordem de aplicação entre vários benefícios. |
| `BenefitEntry` / `benefit_entries` | Confirmado: RF-11 | Casa, período, benefício, valor mensal e referência à configuração efetiva. Um valor por benefício/mês como proposta inicial; zero permitido. |
| `PeriodBenefitPriority` / `period_benefit_priorities` | Proposto se a configuração for copiada para o mês | Benefício mensal, participante do mês e posição. Sem participante repetido nem posições duplicadas na mesma lista. |

Escolher entre regras versionadas referenciadas pelo mês e cópia explícita da prioridade no mês. Não manter duas configurações mutáveis competindo como fonte de verdade. O fechamento sempre copia a ordem efetivamente usada.

### 4.5. Schema `finance`: fechamento e rateio

| Entidade / tabela candidata | Origem | Campos e regras a representar |
| --- | --- | --- |
| `MonthlyPeriodVersion` / `monthly_period_versions` | Confirmado: RF-03/RF-14 e HLD | `id`, casa, período, número do fechamento, versão de edição de origem, autor/data, versão do formato do snapshot e conteúdo histórico do mês. Única por período/número; imutável. |
| `AllocationSnapshot` / `allocation_snapshots` | Confirmado: RF-14 e HLD | Referência única à versão do fechamento, custo total, salários de origem, benefícios de origem, prioridades, percentual, total em dinheiro, benefício aplicado, excedente, total comprometido, diferença, versão do algoritmo e política de arredondamento. |
| `AllocationParticipantSnapshot` | Detalhamento proposto | Por fechamento/participante: identificação histórica, salário, parcela base, benefício abatido e contribuição em dinheiro. Pode ser tabela filha ou conteúdo estruturado do snapshot. |
| `AllocationBenefitApplication` | Detalhamento proposto | Por fechamento/benefício/participante: ordem e valor aplicado, permitindo explicar a distribuição do abatimento. Pode ser tabela filha ou conteúdo estruturado. |

Um fechamento concluído possui exatamente um resultado de rateio. Uma FK única garante no máximo um; a criação obrigatória do par depende da transação de fechamento e das validações correspondentes.

O snapshot mensal precisa preservar despesas, pagamentos, salários, benefícios, prioridades, participantes, nomes relevantes e renda extra da competência. Apenas copiar totais ou guardar FKs para linhas mutáveis não preserva o mês anterior.

**Proposta:** dados operacionais normalizados; snapshot histórico em estrutura versionada, relacional ou `jsonb`, com totais consultados frequentemente em colunas tipadas. A escolha depende das consultas de auditoria e relatórios. Se houver `jsonb`, valores monetários devem continuar com representação decimal exata na serialização e leitura. Não usar o snapshot como substituto das tabelas operacionais.

### 4.6. Schema `income`

| Entidade / tabela candidata | Origem | Campos e regras a representar |
| --- | --- | --- |
| `IncomeSource` / `income_sources` | Confirmado: RF-15 | `id`, casa, nome normalizado, tipo quando necessário, recorrência e estado ativo. Fonte pode ser imóvel alugado; não há requisito de cadastro patrimonial completo. |
| `IncomeEntry` / `income_entries` | Confirmado: RF-15/RF-17 | `id`, casa, período, fonte opcional, tipo recorrente/variável, valor, autoria/datas e observação se adotada. Fonte é exigida para lançamento recorrente; renda variável pode não possuir fonte cadastrada. |
| Sugestão de renda recorrente | Confirmado: RF-16; representação proposta | Valor sugerido, período/lançamento de origem e confirmação. Pode ser estrutura separada, estado explícito no lançamento ou sugestão calculada sem persistir até confirmar. |

Não impor unicidade de fonte/mês a toda `IncomeEntry` antes de decidir se uma fonte aceita vários recebimentos no mês. Rendas variáveis aceitam vários lançamentos explicitamente. Uma sugestão não deve se transformar silenciosamente em renda confirmada; o comportamento dos totais antes da confirmação precisa ser definido.

O período é o mesmo de Finance. Alterações de Income devem respeitar seu estado, versão e bloqueio na mesma transação. A separação de schemas não cria um segundo calendário financeiro.

### 4.7. Auditoria e operações técnicas

| Entidade / tabela candidata | Origem | Campos e regras a representar |
| --- | --- | --- |
| `AuditLog` / `audit_logs` | Confirmado: PRD e HLD | Casa quando aplicável, autor, ação, entidade/ID, período/versão, instante, correlação e metadados controlados. Inserção apenas; consultas restritas. Localização proposta: tabelas por schema/módulo com envelope comum, mantendo os quatro schemas do HLD. |
| `MigrationRecord` / `finance.migration_records` | Confirmado: RF-19 e HLD | Casa, autor, origem, versão do formato, checksum do arquivo, lote, datas, resultado, quantidades e códigos de erro. Não guardar payload bruto sensível em logs. |
| `MigrationItem` / mapeamento de origem | Proposto para deduplicação e rastreabilidade | Migração/lote, identificador estável na origem, tipo de entidade e ID de destino. Permite repetir requisições sem duplicar registros. |
| `ExportRecord` / `finance.export_records` | Confirmado: RF-20 e HLD | Casa, autor, formato/versão, filtros, instante, resultado, contagens e identificador/checksum do conteúdo se necessário. O arquivo é entregue na resposta. |
| `IdempotencyRecord` | Suporte proposto aos requisitos de idempotência | Escopo de módulo/operação, casa quando aplicável, usuário/chave anônima de cadastro, chave idempotente, fingerprint seguro, resultado ou referência estável ao resultado e prazo de retenção. Unicidade no escopo correto e gravação consistente com a operação. |

Cadastro de usuário ocorre sem casa e pode ocorrer sem usuário interno. Não tornar `household_id`/`user_id` obrigatórios para todos os registros técnicos. Uma alternativa simples é separar idempotência global de Identity e idempotência por casa nos demais schemas.

Senhas, tokens e hashes reutilizáveis de senha não devem integrar o fingerprint do cadastro. A deduplicação deve considerar a operação, e-mail normalizado e estado do principal sem persistir o segredo recebido.

Logs operacionais não armazenam valores financeiros sensíveis. Se a auditoria exigir comparação de valores anteriores/posteriores entre edições, definir um registro de revisão de domínio protegido, referenciado pela auditoria, e sua retenção. Um log genérico com apenas ação/ID não reconstrói alterações de rascunho; tampouco os snapshots de fechamento capturam todas essas edições.

## 5. Relacionamentos e cardinalidades

| Relação | Cardinalidade e observação |
| --- | --- |
| `User` → `HouseholdMember` → `Household` | N:N; um usuário pode integrar várias casas. |
| `Household` → membros ativos | 1:N; ao menos um administrador e até dois responsáveis financeiros, além de viewers. |
| `Household` → `MonthlyPeriod` | 1:N, com uma competência por casa. |
| `MonthlyPeriod` → `MonthlyPeriodVersion` | 1:0..N; cada novo fechamento acrescenta uma versão. |
| `MonthlyPeriodVersion` → `AllocationSnapshot` | 1:1 para fechamento concluído. |
| Casa → categoria → categoria no período | 1:N e 1:N; categoria pode existir sem lançamento em determinado mês. |
| `PeriodCategory` → `ExpenseEntry` | 1:0..N. |
| Categoria no período → estado de pagamento | Um estado aplicável quando há lançamento; nunca um pagamento independente por parcela de `ExpenseEntry` neste escopo. |
| Período → participantes → salário | 1:N e 1:0..1; ausência de salário mantém o cálculo pendente quando esse dado for obrigatório. |
| Casa → benefício → valores mensais | 1:N e 1:N; proposta de um valor por benefício/período. |
| Benefício mensal → prioridades → participantes | Lista ordenada de participantes válidos daquele período. |
| Snapshot de rateio → parcelas individuais | 1:N, detalhando responsáveis daquele fechamento. |
| Casa → fonte de renda → lançamentos | 1:N e 1:0..N. |
| Período → lançamentos de renda | 1:0..N; variável pode não referenciar uma fonte. |
| Casa → migrações/exportações/auditoria | 1:0..N; auditoria de identidade pode não pertencer a uma casa. |

O limite de dois responsáveis é uma regra da casa, não motivo para criar colunas `salary_person_1`/`salary_person_2`, campos com nomes de pessoas ou duas FKs fixas no período.

## 6. Invariantes e onde garanti-las

| Regra | Garantia no banco | Garantia no caso de uso/transação |
| --- | --- | --- |
| Referências pertencem à mesma casa | FKs compostas incluindo `household_id` e chaves únicas correspondentes. | Validar vínculo/papel e usar casa autorizada, não somente o ID recebido. |
| Período não se repete | `UNIQUE(household_id, reference_month)`. | Tratar tentativa duplicada/idempotente. |
| Categoria e fonte não se repetem indevidamente | Índice único por casa/nome normalizado, conforme política de reuso. | Validar normalização e apresentar conflito. |
| Lançamentos podem ser múltiplos | PK individual; nenhuma unicidade apenas por categoria/mês. | Usar idempotência para retries, sem impedir lançamentos legítimos de mesmo valor. |
| Valores monetários válidos | Tipo decimal e `CHECK` compatível com sinal/escala aprovada. | Rejeitar formato/escala inválida antes de persistir. |
| Salário/benefício não negativos | `CHECK(amount >= 0)`. | Distinguir valor faltante de zero. |
| Contribuições não negativas | `CHECK` nas parcelas persistidas. | Aplicar benefícios até o limite da parcela; registrar excedente. |
| Pelo menos um administrador e até dois responsáveis | Não se resolve com `CHECK` isolado na linha do membro. | Serializar alterações de membros pela casa; validar contagens dentro da mesma transação. Trigger pode complementar se for uma decisão do projeto. |
| Mês fechado não pode ser editado | Estado e versão persistidos; defesa adicional no banco pode ser definida. | Toda escrita de Finance/Income valida estado, bloqueio e versão de forma atômica. |
| Fechamento exige pagamentos e rateio consistentes | Restrições locais e referências. | Validar todas as categorias com lançamentos, dados obrigatórios e totais antes do commit. |
| Snapshot imutável | Unicidade da versão e restrição de UPDATE/DELETE para o acesso comum, com mecanismo a definir. | Fechar insere nova versão; nunca atualiza uma versão anterior. |
| Prioridade válida | Unicidade de posição e participante por configuração; FKs para participantes. | Verificar lista completa, elegibilidade e ordem determinística entre benefícios. |
| Convite usado uma vez | Token único/estado persistido, se o fluxo for adotado. | Aceite e vínculo na mesma transação, incluindo limite da casa e destinatário. |
| Exclusão bloqueia acesso comum | Estado da casa persistido e índices para localizar elegíveis à eliminação. | Todas as operações, inclusive exportação, verificam estado e autorização. |

Exemplo conceitual de integridade: um lançamento da casa A deve referenciar uma categoria do período da casa A. A FK para o ID da categoria, sozinha, verifica existência, mas não o pertencimento à casa. A mesma disciplina vale para salário/membro, benefício/prioridade e renda/fonte/período.

Mesmo FKs compostas não autorizam acesso: um usuário de outra casa pode tentar fornecer um conjunto inteiro de IDs coerentes. A autorização por vínculo continua obrigatória. Avaliar Row-Level Security como defesa adicional; não há decisão de RLS na documentação atual.

## 7. Estados, versões e transações

### 7.1. Período mensal

Modelo inicial compatível com os documentos: `draft` e `closed`.

1. Abrir: cria período `draft`, versão inicial de edição e configurações aplicáveis.
2. Editar: valida casa/papel, estado, versão esperada e bloqueio; grava e incrementa a versão.
3. Fechar: valida todos os dados, calcula rateio, cria versão histórica e snapshot, altera estado para `closed` e registra auditoria em uma transação.
4. Reabrir: registra autor/data, retorna para `draft` e incrementa a versão de edição, preservando fechamentos anteriores.
5. Fechar novamente: cria o próximo número de fechamento, sem sobrescrever versões anteriores.

`edit_version` e `closing_number` têm finalidades diferentes. A primeira muda nas edições; o segundo cresce somente a cada fechamento concluído. A reabertura precisa de evento próprio, pois os campos do último fechamento não contam toda a história.

Um ponteiro opcional para a última versão deve referenciar uma versão do próprio período/casa. Mantê-lo exige integridade e atualização na transação de fechamento.

### 7.2. Bloqueio temporário

- Persistir titular, identificação da aquisição e `expires_at`.
- Aquisição/renovação/liberação condicionais no banco; chamadas concorrentes não podem obter simultaneamente o mesmo bloqueio.
- Cinco minutos de validade, renovados por atividade REST conforme o HLD.
- Usar relógio consistente do servidor/banco. A validade é consultada em cada operação; expiração não depende de limpeza periódica.
- Identificador da aquisição evita que uma aba antiga libere um bloqueio adquirido depois. A política para várias abas do mesmo usuário permanece a definir.
- Renovação do bloqueio pode ter controle separado para não gerar conflitos de conteúdo a cada heartbeat.
- O fechamento e a reabertura também precisam disputar a mesma unidade de concorrência das alterações de dados.

O bloqueio de edição é uma concessão com prazo persistida em dados; não se deve manter uma transação SQL aberta por cinco minutos. Transações curtas protegem cada operação. Bloqueio lógico não substitui a versão otimista.

### 7.3. Transações necessárias

| Operação | Conteúdo atômico |
| --- | --- |
| Criar casa | Casa, primeiro administrador e registro de auditoria. |
| Alterar membros/aceitar convite | Validação serializada dos limites, vínculo/papel, estado do convite quando aplicável e auditoria. |
| Salvar dados do mês | Verificação/avanço condicional da versão do período, lançamento/configuração/pagamento e auditoria. |
| Salvar renda extra | Mesma proteção de período de Finance, mais escrita no schema Income. |
| Fechar mês | Validação, snapshot completo de uma visão consistente, rateio, estado, versões, auditoria e resultado idempotente. |
| Reabrir mês | Estado, versão e evento de reabertura; versões históricas permanecem intactas. |
| Confirmar lote de migração | Importação do lote e resultado/mapeamentos consistentes; definir se atomicidade será por lote ou arquivo. |
| Excluir/restaurar casa | Estado, prazos e auditoria, coordenados com escritas simultâneas. |

Verificar o estado da casa antes de uma transação e depois escrever sem coordenação permite uma corrida com a exclusão. Definir proteção transacional comum também para esse caso.

Cognito e PostgreSQL não compartilham transação. Cadastro exige a compensação descrita no FDD e a recuperação idempotente por `cognito_sub`/e-mail; uma transação local isolada não resolve falhas entre os dois sistemas.

## 8. Rateio, precisão e dados históricos do cálculo

Confirmado no PRD RF-10 a RF-14:

- Salários líquidos mensais dos responsáveis.
- Menor percentual inteiro capaz de cobrir o custo.
- Benefícios abatem contribuições conforme prioridade.
- Contribuições em dinheiro não podem ser negativas.
- Identificação de benefício excedente.
- Persistência dos valores de origem e do resultado a cada fechamento.

O PRD não fornece fórmula algébrica completa nem política de arredondamento. A POC fornece a seguinte **referência de comportamento**, que precisa ser ratificada para o backend:

```text
C = soma dos custos do mês
S = soma dos salários líquidos
p = teto(100 × C / S)              // percentual inteiro, para S > 0 e C > 0
parcela_base_i = salário_i × p / 100

Para cada benefício, em ordem definida:
  aplicar até o saldo da parcela do primeiro participante da prioridade;
  continuar nos demais participantes;
  registrar o benefício restante como excedente.

contribuição_em_dinheiro_i = parcela_base_i - benefício_aplicado_i
total_comprometido = soma(contribuições_em_dinheiro) + benefícios_aplicados
diferença = total_comprometido - C
```

Na POC o percentual é calculado sobre o custo integral **antes** do abatimento. Renda extra não compõe esse cálculo. Benefício excedente não é transferido automaticamente para o mês seguinte. Essas observações não substituem a confirmação da regra final.

Exemplo hipotético para discutir a regra: custo de R$ 1.000, salários de R$ 2.000 e R$ 3.000, benefício de R$ 500 priorizando o primeiro participante. O percentual é 20%; parcelas base de R$ 400/R$ 600; benefício aplicado de R$ 400/R$ 100; contribuições em dinheiro de R$ 0/R$ 500; total de R$ 1.000 e diferença zero. Esse exemplo não utiliza valores reais da casa.

Decisões de cálculo que afetam o banco e seus contratos:

- `numeric(18,2)` é uma proposta para valores monetários finais; confirmar limite de valores, moeda e escala. Cálculos intermediários precisam preservar precisão antes do arredondamento final.
- Definir arredondamento de meio centavo, estágio de arredondamento e distribuição de resíduos entre participantes. .NET e PostgreSQL devem aplicar a mesma política explicitamente.
- Definir como garantir cobertura após arredondar parcelas. O menor percentual com cálculo exato pode exigir ajuste na reconciliação em centavos.
- Percentual de comprometimento é inteiro em pontos percentuais; não misturar armazenamento `20` com fração `0,20`.
- Não impor teto de 100% sem decisão: custos podem superar a soma dos salários, e a POC admite percentual superior a 100.
- Custo zero, salários ausentes, ambos os salários zero e benefício suficiente com salário zero precisam de regras próprias; não inferir comportamento do fallback numérico da POC.
- Definir se é permitido fechar com somente um responsável e se benefício sem valor significa zero ou pendência.
- Definir sinal permitido para despesas/rendas e tratamento de estornos. A documentação proíbe explicitamente salários/benefícios negativos, mas não especifica o modelo de devoluções.

O snapshot deve guardar versão do algoritmo, moeda/escala, política de arredondamento, valores informados, prioridades efetivas, parcelas base, abatimentos por benefício/participante, excedentes e resultados finais. Consultar um fechamento antigo deve devolver o resultado armazenado, mesmo que o algoritmo atual tenha mudado.

## 9. Convenções físicas, índices e consultas

As definições abaixo são propostas para a modelagem física, não decisões existentes no HLD.

| Tema | Proposta e decisão necessária |
| --- | --- |
| PK | `uuid` gerado pela aplicação como candidato; alinhar com o ID público textual do FDD. Não usar e-mail/nome como PK. |
| Competência | `date` representando o primeiro dia do mês, com validação. Alternativa: ano/mês separados com restrições. |
| Instantes | `timestamptz`, trabalhando com instantes UTC; fuso de apresentação não deve alterar a competência. |
| Monetários | `numeric`/`decimal`; confirmar precisão/escala e validar entrada antes de arredondamento implícito. Evitar tipos de ponto flutuante. |
| Versão de edição | Inteiro crescente, por exemplo `bigint`, com atualização condicional. |
| Status/papéis | Conjunto fechado via `CHECK` ou enum; escolha uniforme nas migrations. |
| Nome/e-mail | Valor de exibição separado de valor normalizado para unicidade; decidir espaços, caixa, acentos e limites. |
| Texto livre | Tamanho máximo para nomes e observações definido em contrato; evitar payloads arbitrários. |
| Snapshot | Conteúdo versionado em tabelas filhas e/ou `jsonb`, com estratégia de leitura de versões antigas. |
| Remoção | Política por entidade; `RESTRICT` nas relações históricas de uso comum, eliminação controlada da casa após prazo. |

Índices candidatos, além das PKs:

| Consulta/regra | Chave candidata |
| --- | --- |
| Login/perfil | Únicos em `cognito_sub` e `normalized_email`. |
| Casas do usuário | Membros por `user_id` e estado; unicidade de vínculo ativo por casa/usuário. |
| Competências de uma casa | Único em `(household_id, reference_month)`. |
| Categorias e fontes por nome | Único em `(household_id, normalized_name)`, com política explícita para inativos. |
| Categoria no mês | Único em `(household_id, monthly_period_id, expense_category_id)`. |
| Lançamentos da categoria no mês | `(household_id, period_category_id)`. Se período/categoria forem colunas diretas, indexar essa composição. |
| Salário/benefício mensal | Único por casa/período/participante e por casa/período/benefício, respectivamente. |
| Histórico de renda | `(household_id, income_source_id, monthly_period_id)` e acesso por casa/período. |
| Versões de fechamento | Único em `(household_id, monthly_period_id, closing_number)` e FK única para o rateio. |
| Auditoria | Casa/instante e, conforme demanda, casa/tipo/ID da entidade. |
| Eliminação de casas | Prazo de recuperação das casas excluídas. |
| Login, idempotência e convites | Índices de chave/escopo e expiração compatíveis com a política adotada. |

Definir primeiro as queries e FKs finais; evitar índices redundantes e validar planos com volume representativo. Índices são parte da estratégia para a meta do HLD de `p95 < 150 ms`, mas não garantem a meta isoladamente.

Consultas a detalhar antes da modelagem física:

- Mês completo: categorias, lançamentos, pagamentos, participantes, salários, benefícios, renda extra, estado e versão.
- Histórico de categoria, incluindo zero em meses sem lançamento sem fabricar registros financeiros zero no banco.
- Custo total/pago/pendente, maiores despesas e evolução mensal.
- Último valor conhecido de uma fonte ativa antes da competência aberta; não copiar de uma competência posterior.
- Renda total, aluguéis e renda variável por mês.
- Histórico de fechamentos e consulta de uma versão específica.
- Auditoria, resumo de importação e exportação consistente de uma casa.

Totais e médias de rascunho podem ser calculados por consulta. Não são necessárias tabelas para cada gráfico. Para histórico fechado, definir se a consulta usa o último fechamento ou uma versão escolhida; meses reabertos devem distinguir estado atual de último fechamento.

A janela da média, inclusão de rascunhos, meses sem movimento e período anterior à criação de uma categoria/fonte não estão definidos. A POC calcula certas médias usando somente registros existentes; isso não resolve a semântica final dos relatórios.

## 10. Migração e exportação

### 10.1. Dados disponíveis e limites do legado

O README da POC documenta as abas originais `Custos`, `Renda Extra` e `Cartão da Casa`, além das abas geradas `POC - Custos` e `POC - Renda Extra`. O código de exportação escreve resumos mensais e seções de despesas, aluguéis e renda variável. Antes de construir o importador, será necessário conferir a planilha efetiva e escolher a fonte autoritativa entre abas originais, abas da POC e estado salvo no navegador.

| Dado legado documentado ou observado no código | Destino candidato | Cuidado necessário |
| --- | --- | --- |
| Nome de despesa | `ExpenseCategory` | Normalizar nomes e reconciliar duplicidades; não inferir que nomes parecidos representam a mesma categoria. |
| Valor de despesa por mês | Um `ExpenseEntry` agregado de origem legada | A POC armazena um total por categoria/mês; não inventar a composição dos lançamentos originais. |
| Campo `paid` | Estado da categoria no período | O estado inicial da POC marca automaticamente como pagos os registros anteriores ao mês de referência. Essa convenção não comprova pagamento real. |
| Salários de pessoas identificadas nominalmente | `PeriodParticipant` e `SalaryEntry` | Mapear explicitamente cada pessoa para um usuário/membro; não criar contas ou vínculos por suposição. |
| Ticket | `Benefit`, valor mensal e prioridade | A POC tem uma prioridade fixa entre duas pessoas. Confirmar sua validade para cada competência importada. |
| Nome de imóvel e aluguel mensal | `IncomeSource` e `IncomeEntry` | Preservar a fonte estável sem exigir um cadastro patrimonial fora do escopo. |
| Total de renda variável mensal | Um lançamento agregado de renda variável | Não há evidência suficiente para reconstruir dividendos ou recebimentos individuais. |
| Expressão informada | Metadado de origem opcional | Não executar expressão arbitrária nem usá-la como prova de vários lançamentos independentes. |
| Listas separadas de meses de custos/renda | `MonthlyPeriod` compartilhado | Reconciliar competências sem duplicar mês por tela; ausência em uma lista não equivale a zero confirmado. |
| Resumos e percentuais calculados | Valores de controle da importação | Conferir contra os detalhes; não somar resumos novamente como lançamentos. |
| Aba `Cartão da Casa` | Sem importação detalhada prevista | Gestão detalhada de cartão está fora do MVP. Decidir apenas se existe algum total já incluído nas despesas, evitando duplicidade. |

Também há limitações de representação: a POC usa ponto flutuante e o exportador converte alguns valores ausentes/inválidos para zero. A carga inicial contém meses de despesas sem salários completos. Por isso, importar um resultado numérico não significa que haja informação suficiente para validar um fechamento conforme as regras novas.

### 10.2. Fluxo e rastreabilidade necessários

1. Definir formato aberto, versão do contrato, colunas obrigatórias, limites de arquivo/lote e período permitido.
2. Validar antes da confirmação: competência, valor, escala, nome, duplicidade, pessoa/fonte e estado de pagamento. Apresentar erros por registro sem gravar dados financeiros.
3. Vincular a confirmação ao mesmo conteúdo validado, por checksum e contexto de casa/operação, e revalidar conflitos no momento da gravação. A base pode mudar entre prévia e confirmação.
4. Aplicar política explícita para registros existentes: rejeitar, conciliar ou atualizar sob controle. Não sobrescrever automaticamente meses fechados.
5. Persistir origem e mapeamento suficiente para reconhecer registros repetidos. Mesma categoria e mesmo valor não são, isoladamente, uma chave de deduplicação segura.
6. Confirmar a importação e registrar seu resultado dentro da requisição. Dividir volumes grandes em lotes REST idempotentes, respeitando o limite transacional definido.
7. Reconciliar totais por categoria/período e por fonte/período, pagamentos, salários, benefícios e resultados legados. Divergências devem ser visíveis antes da conclusão do processo de migração.

Um lote concluído não deve ser anunciado como arquivo inteiro concluído se houver outros lotes pendentes. Uma falha deve deixar claro o que foi confirmado e o que foi revertido. O registro de falha pode exigir gravação separada da transação financeira revertida; ele não pode sobreviver afirmando que dados não confirmados foram importados.

Meses antigos sem evidências suficientes podem entrar como rascunho para revisão, como proposta inicial. Se o produto exigir estado histórico importado distinto, isso precisa ser definido antes de criar o enum de período. Não atribuir datas de fechamento, autores ou pagamentos retroativos fictícios; data de importação é diferente da data do fato original.

### 10.3. Exportação consistente

- Definir se o arquivo contém estado atual, todas as versões de fechamento, auditoria e dados necessários para reimportação.
- Incluir versão do formato, identificadores estáveis, competência, moeda/escala e instante de geração.
- Gerar o conteúdo a partir de uma visão consistente dos dados; uma exportação não pode misturar salários anteriores com despesas alteradas durante a leitura.
- Validar associação, papel administrativo e estado da casa, inclusive após retries.
- Concluir a geração na requisição e registrar seu resultado. Sucesso na geração/envio pelo servidor não comprova que o usuário salvou o arquivo no dispositivo.
- Não há requisito de armazenar permanentemente o arquivo exportado no banco. Se houver retenção temporária, definir localização, acesso e prazo.

## 11. Exclusão, recuperação, retenção e operação

### 11.1. Ciclo de vida da casa

O HLD determina exclusão lógica, recuperação por 30 dias e eliminação definitiva via endpoint administrativo após o prazo. Para cumprir esse fluxo, o modelo e as operações precisam:

- Registrar quem excluiu, quando excluiu e o limite de recuperação; decidir se o prazo é contado como duração exata ou por calendário.
- Bloquear imediatamente o acesso comum a todos os dados da casa, inclusive consultas diretas a filhos, migrações e exportações.
- Definir quem pode recuperar a casa e como acessa esse fluxo excepcional durante o bloqueio comum.
- Preservar categorias, membros, lançamentos e snapshots durante a janela de recuperação.
- Coordenar exclusão, restauração, novas escritas e eliminação definitiva para que uma corrida não reative dados parcialmente removidos.
- Executar a eliminação somente após validar prazo e autorização operacional; definir limites e eventual divisão explícita de uma casa muito grande em operações REST.
- Incluir no inventário de eliminação dados financeiros, snapshots, vínculos, convites, auditorias/revisões e arquivos temporários, respeitando a política de retenção que vier a ser definida.

**A exclusão de uma casa não equivale à exclusão do usuário.** Um usuário pode participar de várias casas. Evitar cascatas de `Household` para `User` ou exclusão do principal Cognito por consequência automática da remoção de uma casa. Exclusão de conta é outro fluxo, ainda não detalhado na documentação.

A imutabilidade do fechamento protege o histórico durante o uso normal. Ela precisa coexistir com a eliminação definitiva autorizada; o acesso comum e o mecanismo administrativo de remoção devem ter capacidades distintas.

### 11.2. Retenção a especificar

| Conjunto de dados | O que falta definir |
| --- | --- |
| Dados financeiros e snapshots | Prazo de retenção em casas ativas e tratamento após exclusão. |
| Auditoria e revisões | Conteúdo, acesso, prazo e política de pseudonimização/eliminação. |
| Tentativas de login e bloqueios | Janela de detecção, tempo necessário após desbloqueio e limpeza dos registros expirados. |
| Idempotência | Janela de retry, retenção do resultado e comportamento após expiração da chave. |
| Convites | Expiração, reenvio, retenção de aceites/cancelamentos e eliminação do e-mail/token derivado. |
| Reconciliação de identidade | Responsável operacional, estados, encerramento e retenção de evidências. |
| Migração/exportação | Retenção dos registros, relatórios de erro, arquivos de entrada e arquivos temporários. |
| Backups e logs operacionais | Retenção por ambiente e procedimento para reaplicar exclusões após restauração. |

O HLD deixa a política de retenção e exclusão dependente de validação jurídica. Este levantamento registra essa dependência documental; não determina prazos legais.

Expiração lógica e remoção física são operações diferentes. Bloqueios e convites vencidos devem perder validade mesmo que suas linhas ainda existam. Como não há scheduler na aplicação, definir procedimento operacional para limpeza e eliminação via meios compatíveis com o HLD, sem criar silenciosamente um serviço de jobs.

Backups são uma capacidade de infraestrutura, não uma tabela de domínio. O HLD estabelece metas de RPO de até cinco minutos e RTO de até 60 minutos, sujeitas à validação. O procedimento de recuperação precisa identificar e reaplicar exclusões ocorridas depois do ponto restaurado; falta definir onde manter essa evidência sem reter desnecessariamente os dados eliminados.

## 12. Decisões pendentes antes da modelagem física

Os itens abaixo não impedem concluir este levantamento. Eles indicam escolhas que precisam ser resolvidas antes de consolidar constraints, entidades e migrations de cada módulo.

| ID | Decisão | Efeito no modelo e encaminhamento proposto |
| --- | --- | --- |
| D01 | Convite com aceite ou associação direta a usuário existente? | Define a existência de `HouseholdInvitation`, expiração e estados. Resolver a diferença entre RF-01 e o fluxo simplificado do HLD. |
| D02 | Como comprovar o destinatário do convite se não há confirmação de e-mail no cadastro? | Aceite precisa de regra segura de identidade/posse do convite; não presumir que e-mail declarado e e-mail confirmado são equivalentes. |
| D03 | Administrador é sempre responsável financeiro? Quantos administradores são permitidos? | Define se papel e participação financeira são separados; o limite de dois deve contar os participantes corretos. |
| D04 | É possível operar/fechar com apenas um responsável? Como substituir um responsável? | Define validação de completude e vigência de participantes. Proposta: preservar os participantes de cada período. |
| D05 | Como mudanças de categoria/fonte/membro afetam meses existentes e reabertos? | Escolher vigência, configuração mensal e política de renomeação. Fechamentos existentes sempre permanecem preservados. |
| D06 | Pagamento por categoria confirmado? Alterar valor de categoria paga exige nova confirmação? | HLD estabelece pagamento por categoria/mês. Definir se alterações monetárias retornam o estado para pendente; proposta: exigir nova confirmação. |
| D07 | Qual o significado de despesa zero, ausência e valores negativos? | Define `CHECK`, nulabilidade, pendência de pagamento e eventual modelo de estorno. Salário/benefício negativo já é proibido. |
| D08 | O que torna uma despesa recorrente? | RF-04 permite marcar categoria recorrente, mas não define cópia de valor, agendamento ou vencimento. Proposta: limitar inicialmente à presença/sugestão no novo mês, após confirmação da regra. |
| D09 | Benefício pertence a um membro? Como ordenar múltiplos benefícios e prioridades? | Define titularidade, regras ordenadas e vigência; preservar configuração usada em cada mês. |
| D10 | Benefício excedente expira ou passa para outro mês? Renda extra entra no rateio? | A POC não carrega saldo e não inclui renda extra. Confirmar antes de criar saldo/crédito ou alterar bases do cálculo. |
| D11 | Qual fórmula final e política de centavos? | Ratificar cálculo da POC ou especificar outro, arredondamento, resíduos, custo zero, salário total zero e percentual superior a 100%. |
| D12 | Qual moeda, escala, limites e padrão de identificadores? | Fecha tipos físicos, contratos e validações. Propostas iniciais: moeda única BRL, `numeric(18,2)` e UUID interno, sujeitas a confirmação. |
| D13 | Uma fonte recorrente admite vários lançamentos no mês? Como confirmar sugestões? | Define unicidade e separação entre sugestão e renda confirmada. |
| D14 | O fechamento inclui a renda extra? Como consultar um mês reaberto? | Proposta: período compartilhado com snapshot completo; distinguir dados em edição do último fechamento. Confirmar recorte dos relatórios. |
| D15 | Quais dados integram médias e gráficos? | Definir janela, meses zero, rascunhos, período de existência da categoria/fonte e versão histórica utilizada. |
| D16 | Como serão persistidos os snapshots e as revisões? | Escolher tabelas filhas/`jsonb`, conteúdo, nomes históricos e extensão da auditoria de valores antes/depois. |
| D17 | Qual política de remoção de lançamentos e reuso de nomes? | Define exclusão lógica/física, índices únicos para inativos, filtros e histórico. |
| D18 | Qual fonte histórica prevalece e quais dados são confiáveis? | Conferir planilhas/POC; definir pagamentos, autores, períodos incompletos, estado importado e tratamento de divergências. |
| D19 | Qual contrato e limite de importação/exportação? | Define lotes, atomicidade, validação/confirmação, formato aberto, deduplicação e recorte exportado. |
| D20 | Qual escopo/validade de idempotência? | Definir retry concorrente, chave repetida com conteúdo diferente, falha entre Cognito/banco e resultado de operação incompleta. |
| D21 | Limite de login usa par e-mail/IP ou limites independentes? Janela fixa ou móvel? | O FDD não resolve a composição das chaves. Define tabela de contadores/eventos; preservar 10 falhas em 15 minutos e bloqueio por 15 minutos conforme contrato aprovado. |
| D22 | Qual a política de retenção, recuperação e eliminação? | Define prazos, autorização de restauração, ordem de remoção, evidência de exclusões e tratamento de backups. |
| D23 | Como funcionam múltiplas abas e tomada de bloqueio? | Define titular da concessão, identificador de aquisição, renovação e liberação de edição. |
| D24 | Qual defesa e organização física adicional serão usadas? | Decidir RLS, proteção de snapshots no banco, um ou mais DbContexts, localização da auditoria e acesso administrativo para eliminação. |

As decisões com maior impacto estrutural são D01, D03–D05, D09, D12–D14, D16 e D18. D06–D11 exigem também exemplos de negócio antes da implementação do cálculo e fechamento. As demais precisam estar resolvidas antes de finalizar a implementação dos respectivos fluxos.

## 13. Validações que o modelo deve permitir

Este é um roteiro para os testes futuros, não uma suíte implementada nesta etapa.

| Cenário | Resultado esperado |
| --- | --- |
| Usuário tenta acessar ou exportar outra casa | Operação recusada independentemente de conhecer IDs válidos. |
| Lançamento referencia categoria/fonte/membro de outra casa | Integridade referencial impede a combinação. |
| Duas requisições abrem a mesma competência | Um período persistido; resposta duplicada/conflito conforme contrato. |
| Duas inclusões simultâneas excederiam dois responsáveis | Limite preservado na transação. |
| Duas alterações tentam remover os últimos administradores | Casa mantém administrador enquanto ativa. |
| Duas edições usam a mesma versão | Uma confirma; a outra recebe conflito sem sobrescrita silenciosa. |
| Bloqueio expira e outro editor o adquire | Editor antigo não edita/libera usando a aquisição anterior. |
| Edição de renda compete com fechamento | Snapshot consistente e proteção pelo mesmo período. |
| Fechamento contém categoria com valor informado pendente | Fechamento recusado; categoria sem lançamento não impede por si só. |
| Novo lançamento é adicionado à categoria paga | Aplica política de reconfirmação definida em D06. |
| Reabrir e fechar novamente | Nova versão; conteúdo da primeira versão permanece igual. |
| Categoria é renomeada/desativada ou membro substituído | Fechamento antigo mantém dados/resultados históricos. |
| Salário ausente comparado a salário zero | Validação distingue ausência de zero legítimo. |
| Benefício supera uma parcela ou todas as parcelas | Redistribuição conforme prioridade, contribuições não negativas e excedente explícito. |
| Rateio produz frações de centavo | Reconciliação determinística segundo política aprovada. |
| Sugestão recorrente é reapresentada/reconfirmada | Não duplica renda; origem e confirmação são preservadas. |
| Retry repete importação ou lançamento | Resultado idempotente, sem duplicação nem bloqueio de lançamento legítimo distinto. |
| Cognito cria principal e persistência local falha | Compensação/reconciliação conforme FDD, sem perfil inconsistente autorizado. |
| Tentativas inválidas ocorrem em réplicas distintas | Limite de login compartilhado e bloqueio consistente. |
| Exclusão compete com escrita/restauração | Estado e dados permanecem coerentes; acesso comum bloqueado durante exclusão. |
| Casa eliminada tinha usuário em outra casa | Dados e conta necessários à outra casa são preservados. |
| Restauração de backup anterior a uma exclusão | Exclusões são reaplicadas antes de disponibilizar dados aos usuários. |

## 14. Ordem sugerida para a próxima etapa

1. Resolver as decisões de negócio que mudam cardinalidades, vigência, pagamentos e cálculo; registrar os critérios escolhidos nos documentos de origem.
2. Elaborar o modelo conceitual e o diagrama de entidades e relacionamentos por módulo, incluindo os vínculos entre schemas.
3. Produzir o dicionário definitivo: campos, tipos, nulabilidade, valores permitidos, PKs, FKs, uniques, checks, índices e políticas de remoção.
4. Definir o conteúdo versionado dos snapshots, a auditoria e as transações dos fluxos críticos.
5. Implementar entidades/regras em `Expenses.Domain`, casos de uso em `Expenses.Application` e mapeamentos/repositórios/migrations em `Expenses.Infrastructure`. DTOs de request/response não determinam automaticamente as tabelas.
6. Validar migrations e restrições em PostgreSQL, cobrindo isolamento, concorrência, rateio, histórico e exclusão. Depois implementar e validar o importador com uma cópia dos dados reais.

O levantamento está concluído como inventário de requisitos e decisões. A modelagem física final depende das escolhas explicitadas, especialmente sobre histórico, participação financeira e dados legados.
