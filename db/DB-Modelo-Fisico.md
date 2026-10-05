# Modelo físico do banco de dados — sprint-1

Data: 2026-10-04. PostgreSQL 17 e Liquibase YAML/SQL.

## Escopo e fontes

A [sprint-1](../expenses-liquibase/changelogs/sprint-1/changelog.yaml) cria 33 tabelas nos schemas `identity`, `household`, `finance` e `income`, com 33 PKs e 88 FKs. Os oito scripts SQL constituem o dicionário de colunas, tipos, nulabilidade, defaults, constraints e índices. Cada tabela tem comentário no catálogo PostgreSQL.

O modelo materializa o [levantamento](Levantamento-Modelagem-Banco-de-Dados.md), o [PRD](PRD.md), o [HLD vigente](HLD.md) e o [FDD de identidade](FDD-Criacao-Usuario-Autenticacao.md). A documentação e o código da POC citados no levantamento orientam a compatibilidade com o legado. A planilha não foi importada nem teve seus dados certificados nesta etapa.

As escolhas abaixo são premissas de implementação da sprint, registradas para revisão; não representam aprovação prévia de todas as decisões de produto. O banco oferece estruturas para os casos de uso, que ainda precisam ser implementados no backend.

## Inventário

| Schema / SQL | Tabelas | Finalidade |
| --- | --- | --- |
| `identity` — [002](../expenses-liquibase/changelogs/sprint-1/sql/002-create-identity.sql) | `users` | Perfil interno, e-mail normalizado único e vínculo Cognito. |
| `identity` — 002 | `login_attempts`, `login_blocks`, `identity_reconciliations` | Falhas de login, bloqueios compartilhados e divergências Cognito/banco. |
| `identity` — 002 | `idempotency_records`, `audit_logs` | Retry e metadados de identidade. |
| `household` — [003](../expenses-liquibase/changelogs/sprint-1/sql/003-create-household.sql) | `households`, `household_members`, `household_invitations` | Casa, ciclo de vida, vínculos históricos, papéis e convites. |
| `household` — 003 | `idempotency_records`, `audit_logs` | Retry e auditoria da casa. |
| `finance` — [004](../expenses-liquibase/changelogs/sprint-1/sql/004-create-finance.sql) | `monthly_periods` | Competência, estado, versão de edição, concessão e último fechamento. |
| `finance` — 004 | `expense_categories`, `period_categories`, `expense_entries` | Cadastro, configuração mensal, pagamento por categoria/mês e múltiplos lançamentos. |
| `finance` — 004 | `period_participants`, `salary_entries` | Responsáveis e salários da competência. |
| `finance` — 004 | `benefits`, `benefit_priorities`, `benefit_entries`, `period_benefit_priorities` | Cadastro e valores, ordem e prioridades efetivos de cada mês. |
| `income` — [005](../expenses-liquibase/changelogs/sprint-1/sql/005-create-income.sql) | `income_sources`, `income_entries`, `idempotency_records`, `audit_logs` | Fontes, rendas recorrentes/variáveis, sugestões, retry e auditoria. |
| `finance` — [006](../expenses-liquibase/changelogs/sprint-1/sql/006-create-closings.sql) | `monthly_period_versions`, `allocation_snapshots` | Snapshot mensal e rateio em relação obrigatória 1:1. |
| `finance` — [007](../expenses-liquibase/changelogs/sprint-1/sql/007-create-operations.sql) | `migration_records`, `migration_items`, `export_records` | Resultados síncronos por lote, mapa de origem e exportações. |
| `finance` — 007 | `entry_revisions`, `idempotency_records`, `audit_logs` | Revisões antes/depois, retry e auditoria financeira. |

Contagem por schema: Identity 6, Household 5, Finance 18, Income 4. O conceito `PaymentStatus` do HLD está representado por `payment_status`, `paid_at` e `paid_by_user_id` em `period_categories`. Indicadores e gráficos são consultas, não tabelas adicionais.

## Convenções e integridade

- Nomes `snake_case`, schemas explícitos e identificadores dentro do limite de 63 bytes. Sufixos curtos distinguem nomes longos de constraints.
- PKs UUID, normalmente com `gen_random_uuid()` nativo; a aplicação também pode fornecer o ID. `allocation_snapshots.id` compartilha a chave da versão do fechamento.
- Dinheiro em `numeric(18,2)`, não negativo e sem `NaN`; moeda BRL. A API precisa validar escala e limites antes da persistência, pois a conversão para essa escala arredonda casas excedentes. Agregados também precisam caber na precisão escolhida.
- Competência `date` finita, no primeiro dia do mês; instantes `timestamptz`. O backend trabalha com instantes UTC e valida o fuso de apresentação.
- Status/papéis com `CHECK`. Sem extensões adicionais. `created_at` tem default e `updated_at` é atualizado por trigger nas tabelas mutáveis.
- Nomes/e-mails originais preservados; normalização gerada com `lower(btrim(...))`, sem remoção de acentos. Alinhar e-mail com a configuração Cognito e manter collation consistente entre ambientes.
- PKs e `UNIQUE` fornecem seus índices. As FKs possuem índices de apoio nas colunas filhas, reutilizando prefixos de índices compostos. Índices parciais atendem vínculos ativos, sugestões e expiração.
- Relações históricas usam `ON DELETE RESTRICT`. O par fechamento/rateio usa `NO ACTION DEFERRABLE INITIALLY DEFERRED` para inserir/remover ambas as linhas na mesma transação. Não há cascata de casa para usuário.

O PostgreSQL não cria automaticamente índices nas colunas filhas de FKs; os scripts os declaram quando necessário. Ver [constraints PostgreSQL 17](https://www.postgresql.org/docs/17/ddl-constraints.html) e [tipos numéricos](https://www.postgresql.org/docs/17/datatype-numeric.html).

Tabelas de uma casa carregam `household_id`. FKs compostas impedem referências a dados de outra casa ou competência: uma despesa referencia `(household_id, monthly_period_id, period_category_id)`; salários e prioridades referenciam participantes do mesmo mês. FKs de autoria apontam para usuários globais; a autorização desses usuários na casa pertence à aplicação.

Configurações mensais preservam participantes, nomes, valores, ordem e prioridades. Alterações de cadastros não se propagam automaticamente para meses existentes. Fechamentos guardam snapshots completos, preservados após reabertura. `edit_version` conta edições; `closing_number` conta fechamentos. O ponteiro `latest_closed_version_id` exige uma versão da mesma casa e competência.

As referências polimórficas `entity_type`/`entity_id` da auditoria/revisões e `target_entity_type`/`target_entity_id` da importação são referências históricas lógicas, deliberadamente sem FK para o alvo, pois sobrevivem à remoção de lançamentos. O backend valida alvo e casa no momento da gravação; as demais relações dessas tabelas possuem FKs reais.

## Premissas e decisões do levantamento

| Decisão | Implementação e responsabilidade pendente |
| --- | --- |
| D01–D02 — Convites | Aceite/cancelamento/expiração e hash de token. Backend comprova posse e identidade do destinatário, assegura uso único e marca convites vencidos como `expired` antes do reenvio. |
| D03 — Papéis | Administrador pode ter ou não posição financeira; `financial_member` exige posição; `viewer` não ocupa posição. Até duas posições ativas e ao menos um administrador por casa ativa. |
| D04 — Participação | Estrutura admite um ou dois participantes por competência/rateio. Produto/backend definem completude de fechamento e substituição. Vínculos antigos são preservados. |
| D05 — Vigência | Cadastros atuais separados de configurações mensais. Identidade/casa/competência do período não podem ser alteradas. |
| D06 — Pagamento | Por categoria/mês. Inclusão, remoção, alteração de valor ou transferência de lançamento tornam pendente o pagamento das categorias afetadas. Alteração isolada de observação preserva o estado. |
| D07 — Valores | Ausência de linha difere de zero informado. Valores negativos são recusados; estorno exige regra própria. |
| D08 — Recorrência | Flag e configuração mensal; não cria lançamento, vencimento ou pagamento automaticamente. |
| D09 — Benefícios | Titular opcional, ordem de aplicação e prioridades 1/2, copiadas para o mês. Backend valida elegibilidade e completude. |
| D10 — Excedente/renda | Excedente explícito, sem saldo transferível. Renda extra integra o snapshot; compatibilidade inicial com a POC mantém renda fora da base do rateio. |
| D11 — Cálculo | Percentual inteiro sem teto de 100. Totais não negativos; benefício aplicado limitado ao disponível; comprometido = dinheiro + benefício aplicado e cobre o custo. Fórmula, centavos, custo/salário zero e arredondamento pertencem ao algoritmo a ratificar no backend. |
| D12 — Tipos | BRL, `numeric(18,2)`, UUID, `date`, `timestamptz`. O formato público `usr_...` requer conversão/contrato na API. |
| D13 — Renda | Vários lançamentos confirmados por fonte/mês e uma sugestão pendente. Origem é da mesma casa/fonte; backend exige origem anterior e confirmada, escolhe o último valor e confirma a mesma linha com idempotência. Totais incluem apenas `confirmed`. |
| D14–D15 — Consultas | Fechamento inclui custos e renda. Reabertura mantém último fechamento. Janelas de médias e seleção entre rascunho/versão histórica são contratos de consulta pendentes. |
| D16 — Histórico | Snapshots JSON versionados e totais tipados. Revisões de valores separadas da auditoria de metadados. Histórico recusa `UPDATE`. |
| D17 — Remoção/nomes | Nomes continuam únicos mesmo inativos; reativar cadastro existente. Remoção de lançamento em rascunho exige revisão transacional e pode ser limitada por referências históricas existentes. |
| D18 — Legado | Autores antigos desconhecidos podem ser nulos. Importador é ator da operação, não autor fictício do fato. Dados incompletos permanecem rascunhos. Nenhuma carga automática. |
| D19 — Importação/exportação | Operações síncronas por lote, checksum e versão. Lote concluído único por casa/origem/arquivo/lote; mapa estável de origem. Exportação JSON/CSV. Layout, limites, prévia e conciliação pertencem ao backend. |
| D20 — Idempotência | Por módulo/casa/ator/operação/chave; Identity tem escopo explícito para cadastro anônimo. Fingerprint de 32 bytes, estado, resultado seguro e expiração. Backend compara conteúdo e coordena retries; prazo configurável. |
| D21 — Login | Eventos permitem janela móvel e hashes HMAC de 32 bytes; bloqueios por e-mail, IP ou par. Backend define composição/atomicidade e aplica 10 falhas em 15 minutos e bloqueio de 15 minutos. Segredo HMAC fica fora do banco. |
| D22 — Retenção | Backend calcula prazo de recuperação de 30 dias e autoriza restauração/eliminação. Demais prazos permanecem pendentes; não há jobs de limpeza criados. |
| D23 — Edição | Membro, UUID da aquisição, expiração e `edit_version`. Backend aplica cinco minutos, renovação e comparação de versão/token. |
| D24 — Defesa física | Schemas, FKs compostas, índices, triggers e revogação de permissões de `PUBLIC` nos schemas/funções. Roles de implantação, RLS e DbContexts não são criados nesta sprint. |

## Snapshots e transações do backend

O snapshot mensal exige objeto JSON com arrays `categories`, `expenses`, `payments`, `participants`, `salaries`, `benefits`, `benefit_priorities`, `income_sources` e `incomes`. Conjuntos vazios usam arrays vazios. O rateio guarda totais tipados, percentual, moeda, versão do algoritmo, política de arredondamento, versão do contrato e arrays de participantes/aplicações. `difference` e `benefit_excess` são colunas calculadas armazenadas.

O banco valida a estrutura externa do JSON e relações entre totais tipados. O backend deve definir/validar o contrato interno versionado: IDs, nomes históricos, entradas, parcelas, abatimentos, prioridades, excedentes e reconciliação com totais. Usar decimal no cálculo e serialização que preserve centavos. Consultar um fechamento antigo usa o resultado armazenado, sem recalcular com algoritmo novo.

O [script 008](../expenses-liquibase/changelogs/sprint-1/sql/008-create-integrity-guards.sql) serializa mudanças de membros e exige administrador ativo por constraint trigger adiada. Casa e primeiro administrador devem ser criados na mesma transação. Transferência de administração é permitida desde que exista administrador ao commit.

Dados mensais e inserção de fechamento bloqueiam a linha da casa para leitura compartilhada e a do período para atualização; recusam casa excluída e exigem período `draft`. Esses triggers não autorizam usuários nem comparam/incrementam a versão de edição.

Para editar, o caso de uso inicia transação, valida casa/papel, adquire locks na ordem casa → período, confere versão e dono/token/validade da concessão, grava alteração/revisão/auditoria/idempotência e incrementa a versão. Tratar conflitos, deadlocks e falhas de serialização explicitamente.

Para fechar, usar os mesmos locks, verificar participantes/salários/benefícios e pagamentos, tratar sugestões, calcular e montar snapshot consistente. Inserir versão e rateio com o mesmo UUID enquanto o período é `draft`; manter as duas FKs adiadas até ambas as linhas existirem. Atualizar ponteiro, estado e versão; confirmar tudo junto. O banco não calcula o rateio nem compara o JSON com as linhas operacionais.

Para reabrir, autorizar administrador, aplicar controle de concorrência, registrar reabertura e mudar para `draft` incrementando versão. Preservar o ponteiro anterior. Novo fechamento recebe novo número/UUID.

Snapshots recusam atualização; exclusão só é permitida após exclusão lógica da casa e término da recuperação. Auditorias/revisões/resultados recusam atualização, mas retenção e exclusão dependem dos privilégios operacionais. O runtime precisa de privilégios mínimos, sem usar o superusuário local `admin`. Schemas/FKs não restringem `SELECT` por usuário: filtros e autorização pertencem à aplicação, ou a RLS futura.

## Exclusão, evolução e validação

Recuperação/eliminação deve bloquear a casa e verificar prazo na mesma transação. Na eliminação definitiva, remover dependentes antes dos pais, respeitando FKs de sugestões e prioridades. Antes de remover fechamentos, zerar o ponteiro e colocar o período em `draft` dentro da transação de eliminação; remover o par rateio/versão com constraints adiadas. Remover períodos, cadastros, vínculos e casa após seus dependentes. Preservar usuários globais. A rotina administrativa e a política de retenção ainda precisam ser implementadas/testadas.

Os rollbacks removem os objetos na ordem inversa, incluindo triggers e FKs cíclicas, sem `CASCADE`. Removem também os dados da sprint; não são um mecanismo de exclusão de casa. Bases compartilhadas evoluem por novos changeSets, preservando checksums dos já aplicados.

O [README](../expenses-liquibase/README.md) documenta conexão local, aplicação, rollback e testes. `tests/validate_schema.py` aplica/reaplica/reverte/recria a sprint num banco temporário do Docker Desktop. Os testes SQL exercitam integridade entre casas/períodos, unicidade, administrador, valores, pagamentos, salários, prioridades, renda, idempotência, fechamento/reabertura/novo fechamento, histórico e índices das FKs.

Essa validação usa dados sintéticos e não aplica as migrações em `expenses`. Não substitui testes do algoritmo, autorização, concorrência HTTP, concessão de edição, importador, eliminação administrativa e planos com volume real.
