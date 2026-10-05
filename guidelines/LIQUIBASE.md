# Guideline de Liquibase e PostgreSQL

[Índice](README.md) · [Princípios comuns](PRINCIPIOS.md) · [Execução do Liquibase](../../expenses-liquibase/README.md) · [Modelo físico](../db/DB-Modelo-Fisico.md)

Base: Liquibase OSS **4.33.0**, PostgreSQL **17**, changelogs YAML e SQL PostgreSQL. O schema atual contém quatro schemas e 33 tabelas, organizados em oito changesets da sprint-1. Os comandos e exemplos deste documento não foram aplicados ao banco como parte da criação dos guidelines.

## 1. Responsabilidade e estrutura

O Liquibase é o **único responsável pela evolução do schema**. A API não deve criar tabelas ou executar migrations EF no startup. Uma mudança persistente deve entregar o SQL, seu registro no changelog, estratégia de recuperação, testes e ajustes de mapeamento/contrato pertinentes.

Changelogs acompanham a história no [Git Flow](GIT-FLOW.md). Promoção seletiva exige validar dependências entre migrations contra a base de produção; presença de uma tabela em dev/hlg não comprova que ela estará disponível na entrega. Reverter uma branch não reverte o banco.

Manter a estrutura existente:

```text
expenses-liquibase/
  db.changelog-master.yaml
  config/
  changelogs/
    sprint-1/
      changelog.yaml
      sql/
      rollback/
  tests/
```

Para uma nova sprint, adicionar seu diretório e um `include` explícito ao final do master, usando `relativeToChangelogFile: true`. Dentro da sprint, ordenar os changesets pelas dependências. Preferir includes explícitos a uma descoberta automática que torne a ordem menos evidente.

Convenção para arquivos novos: `NNN-acao-objeto.sql`, com correspondente em `rollback/`; changeset `sprint-N-NNN`, autor estável e `dbms: postgresql`. O número comunica a sequência ao leitor; a ordem efetiva vem do changelog, não da ordenação numérica do ID.

## 2. Histórico imutável e identidade

Após aplicação compartilhada, **NÃO alterar SQL, ID, autor, ordem histórica ou caminho de um changeset para mudar seu efeito**. Criar uma nova migração corretiva. A identidade inclui ID, autor e caminho; o checksum detecta alterações relevantes. Referências: [changesets na versão 4.33](https://docs.liquibase.com/oss/user-guide-4-33/what-is-a-changeset) e [checksums](https://docs.liquibase.com/oss/user-guide-4-33/what-is-a-changeset-checksum).

Reorganizar documentação não exige mover changelogs já aplicados. Caso uma mudança de caminho seja indispensável, planejar a preservação da identidade registrada usando `logicalFilePath`, conferir `DATABASECHANGELOG` e testar tanto uma base existente quanto uma vazia. Não adicionar um caminho lógico arbitrário esperando que ele reconheça automaticamente o histórico anterior. Referência: [logicalFilePath 4.33](https://docs.liquibase.com/oss/reference-guide-4-33/changelog-attributes/logicalfilepath).

**NÃO usar** `clear-checksums`, `validCheckSum`, edição de `DATABASECHANGELOG`, `runAlways` ou `runOnChange` como correção automática de divergência. Uma exceção operacional precisa de causa conhecida e procedimento específico. Objetos repetíveis só devem adotar `runOnChange` mediante decisão própria; as migrações estruturais do Expenses são incrementais.

A repetição segura do `update` depende do histórico controlado. `IF NOT EXISTS` em todo DDL pode mascarar um objeto com estrutura errada; não é a política de idempotência do projeto.

## 3. Unidade de mudança e transação

Preferir changesets pequenos com uma intenção verificável. A criação de uma tabela com suas constraints, índice necessário e comentário pode formar uma unidade coesa. Alterações independentes ou com riscos operacionais diferentes devem ser separadas. Não dividir nem reescrever os oito changesets históricos apenas para adequá-los a uma nova preferência de tamanho.

Manter `runInTransaction: true` para operações suportadas. O Liquibase controla a transação; arquivos SQL não devem conter `BEGIN`/`COMMIT` próprios. Falha em uma etapa transacional deve impedir o registro de sucesso dessa unidade. Usar `runInTransaction: false` somente quando o comando exigir, isolando a operação e documentando a recuperação de execução parcial. Referência: [runInTransaction 4.33](https://docs.liquibase.com/oss/reference-guide-4-33/changelog-attributes/runintransaction).

`sqlFile` deve declarar caminho relativo, UTF-8 e estratégia de divisão. Para SQL comum, `splitStatements: true`; para funções com corpo PL/pgSQL e delimitadores internos, usar a configuração validada no projeto, como `splitStatements: false`. Testar a unidade real no executor JDBC. SQL externo não oferece rollback automático: declarar o arquivo correspondente. Referência: [sqlFile 4.33](https://docs.liquibase.com/oss/reference-guide-4-33/change-types/sqlfile).

## 4. Exemplo de formato

**Exemplo didático, sem requisito de produto: não registrar esta tabela no master.** Pressupõe os schemas e `household.households` existentes. Demonstra uma tabela de etiquetas fictícias e sua remoção; não define normalização ou regras reais de categorias.

Changelog ilustrativo:

```yaml
databaseChangeLog:
  - changeSet:
      id: example-001
      author: expenses
      dbms: postgresql
      runInTransaction: true
      changes:
        - sqlFile:
            path: sql/001-create-example-tags.sql
            relativeToChangelogFile: true
            encoding: UTF-8
            splitStatements: true
            stripComments: false
      rollback:
        - sqlFile:
            path: rollback/001-create-example-tags.sql
            relativeToChangelogFile: true
            encoding: UTF-8
            splitStatements: true
            stripComments: false
```

SQL de criação ilustrativo:

```sql
CREATE TABLE household.example_tags (
    id uuid NOT NULL DEFAULT gen_random_uuid(),
    household_id uuid NOT NULL,
    name varchar(80) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT pk_example_tags PRIMARY KEY (id),
    CONSTRAINT uq_example_tags_household_name UNIQUE (household_id, name),
    CONSTRAINT ck_example_tags_name CHECK (btrim(name) <> ''),
    CONSTRAINT fk_example_tags_household FOREIGN KEY (household_id)
        REFERENCES household.households (id) ON DELETE RESTRICT
);

COMMENT ON TABLE household.example_tags IS
    'Exemplo didático de formato; não integra o modelo do produto.';
```

O índice da unicidade começa por `household_id` e já oferece suporte à FK deste exemplo. Um segundo índice apenas nessa coluna seria redundante para esse objetivo.

Rollback ilustrativo:

```sql
DROP TABLE household.example_tags;
```

Esse rollback remove os dados da tabela. Ter um arquivo de rollback não torna uma operação destrutiva reversível em termos de informação.

## 5. Convenções PostgreSQL e integridade

### Nomes e tipos

- **DEVE:** usar `snake_case`, schema explícito e nomes consistentes com as tabelas atuais; evitar identificadores com aspas e diferenças apenas de maiúsculas.
- Usar nomes de constraints/índices previsíveis: `pk_`, `fk_`, `uq_`, `ck_`, `ix_`. Respeitar o limite padrão de 63 bytes; abreviar de modo estável ou usar sufixo determinístico, como já ocorre no modelo.
- PKs UUID e `gen_random_uuid()` seguem a convenção atual, sem exigir extensão adicional no PostgreSQL 17.
- Dinheiro usa `numeric(18,2)` com nulabilidade, limites e exclusão de `NaN` adequados ao campo. Não usar `real`, `double precision` ou o tipo `money` como substituição automática.
- Instantes usam `timestamptz`; datas civis/competência usam `date`. Manter o primeiro dia para competência mensal e os checks de validade previstos no modelo.
- `jsonb` é apropriado aos snapshots versionados e metadados definidos; não substituir relacionamentos centrais por JSON por conveniência.

Referências para os limites de nomes e UUIDs: [estrutura lexical](https://www.postgresql.org/docs/17/sql-syntax-lexical.html) e [funções UUID](https://www.postgresql.org/docs/17/functions-uuid.html) do PostgreSQL 17.

`numeric` permite controle de precisão/escala, mas a coerção pode arredondar antes do armazenamento. A API deve validar entradas conforme o contrato; o tipo não decide a política financeira. Referência: [tipos numéricos do PostgreSQL 17](https://www.postgresql.org/docs/17/datatype-numeric.html).

### Constraints e relacionamentos

Toda tabela de negócio DEVE ter PK. Obrigatoriedade requer `NOT NULL`; `CHECK` sozinho pode aceitar resultado nulo. Usar unicidade para invariantes persistentes e FKs para referências. PK/UNIQUE criam índices de suporte; FKs não criam automaticamente índice nas colunas filhas. `CHECK` não deve consultar outras linhas/tabelas. Referência: [constraints do PostgreSQL 17](https://www.postgresql.org/docs/17/ddl-constraints.html).

No Expenses, preservar FKs compostas que comprovam casa e competência, não apenas a existência do UUID. Uma categoria de outra casa não pode ser associada a um lançamento só porque seu ID existe. A chave candidata referenciada precisa cobrir as mesmas colunas; as constraints devem refletir a relação real.

Definir `ON DELETE` e nulabilidade por ciclo de vida. Manter `RESTRICT` quando apagar o pai violaria histórico; `CASCADE` só mediante relação de propriedade e análise explícitas. Não usar `CASCADE` no rollback para esconder dependências não compreendidas.

Constraints adiadas são exceções justificadas, como o par versão/rateio do fechamento. Documentar a invariância que deve ser verdadeira no commit. Triggers devem ser pequenos, previsíveis e testados; não mover autorização HTTP ou algoritmo financeiro para triggers. As proteções existentes de histórico continuam obrigatórias.

## 6. Índices e custo de consulta

Cada índice novo deve indicar consulta ou integridade que atende. Verificar seletividade, ordem das colunas, filtros e ordenação usados. Inspecionar índices existentes antes de criar outro; um índice composto pode atender a FK quando sua parte inicial é adequada. O teste atual exige cobertura de índices das FKs: preservar o critério ou justificar uma revisão específica.

Avaliar planos com volume representativo. `EXPLAIN ANALYZE` executa a instrução; usar apenas consultas e ambiente apropriados para a análise. Índices também custam escrita, armazenamento e manutenção. Índice parcial deve corresponder ao predicado real; não propor GIN para todo `jsonb` nem particionamento sem evidência. Referências: [índices compostos](https://www.postgresql.org/docs/17/indexes-multicolumn.html) e [Using EXPLAIN](https://www.postgresql.org/docs/17/using-explain.html).

**Para bases povoadas:** avaliar `CREATE INDEX CONCURRENTLY` quando o bloqueio de escrita do índice convencional for inadequado. Esse comando não pode rodar dentro de bloco transacional, pode demorar mais e deixar índice inválido após falha. Isolá-lo em changeset não transacional com uma única instrução e procedimento de diagnóstico/recuperação; não misturar `SET` ou outros DDL no mesmo arquivo. Referência: [CREATE INDEX no PostgreSQL 17](https://www.postgresql.org/docs/17/sql-createindex.html).

Não aplicar `CONCURRENTLY` indiscriminadamente à criação inicial de tabelas vazias.

## 7. Evolução compatível e locks

**RECOMENDADO:** usar expansão → migração dos dados → contração quando uma alteração incompatível alcançar ambiente compartilhado:

1. Adicionar estrutura compatível com a versão antiga e a nova da aplicação.
2. Publicar código capaz de conviver com ambas, quando necessário.
3. Preencher/transformar dados com execução limitada, observável e repetível de forma controlada.
4. Verificar integridade, completude e uso do novo contrato.
5. Remover a estrutura antiga em outra entrega, após encerrar o período de compatibilidade.

Procedimentos operacionais de migração de dados não autorizam introduzir jobs de negócio no MVP. Planejar o mecanismo e a janela de execução sem criar processamento invisível no startup da API.

Para cada DDL em tabela povoada, analisar lock, varredura, possível reescrita, volume e duração. Definir limites de espera/execução conforme a operação; em changesets transacionais, `SET LOCAL lock_timeout`/`statement_timeout` pode limitar o impacto, com valores justificados. Não manter timeout infinito por conveniência.

No PostgreSQL 17, FKs e checks podem ser adicionados como `NOT VALID` e validados depois. Novas escritas continuam sujeitas à constraint, mas dados antigos precisam de `VALIDATE CONSTRAINT`. Isso reduz alguns bloqueios prolongados; não elimina locks nem validação. Não usar esse procedimento para PK/UNIQUE como se tivesse a mesma sintaxe. Referência: [ALTER TABLE no PostgreSQL 17](https://www.postgresql.org/docs/17/sql-altertable.html).

Alterar coluna para `NOT NULL`, converter tipo ou renomear campo consumido exige uma estratégia específica. Não presumir que um DDL curto é operacionalmente barato.

## 8. Preconditions, drift e recuperação

Usar preconditions para premissas relevantes: banco esperado, objeto dependente ou dados que permitem uma transformação. Para pré-condições essenciais, adotar `onFail: HALT` e `onError: HALT`. Não usar `MARK_RAN` para tratar uma diferença desconhecida entre schema e histórico. Referência: [preconditions 4.33](https://docs.liquibase.com/oss/user-guide-4-33/what-are-preconditions).

Checks de dados antes de um DDL não substituem constraints/locks diante de escritas concorrentes. Testar o intervalo entre verificação e alteração quando houver disputa possível.

Em uma falha, verificar alvo, logs, histórico e estado real dos objetos antes de repetir. O lock do Liquibase serializa migradores, mas não interrompe escritores da aplicação. Só liberar lock de execução após confirmar que não há migrador ativo. Não apagar volume, recriar `expenses` ou editar tabelas de controle como procedimento padrão de recuperação.

## 9. Rollback e dados

Toda mudança DEVE declarar se a reversão é segura, destrutiva ou se exige restauração/correção futura. O rollback precisa desfazer dependências na ordem inversa. Para funções/triggers alterados, pode ser necessário restaurar a definição anterior, não apenas excluir o objeto.

Priorizar nova migração corretiva quando houver dados a preservar. `DROP COLUMN`, redução de precisão ou transformação com perda de informação não recuperam conteúdo com um simples SQL inverso. Documentar backup, restauração, compatibilidade da aplicação e janela de recuperação quando aplicáveis.

Não reutilizar `rollback-count --count=8` como comando permanente: oito é a quantidade histórica da sprint-1. Novas migrações alteram o que esse número desfaria. Ensaiar a reversão da entrega em banco isolado com dados representativos antes de considerar seu uso operacional.

## 10. Validação e entrega

Com a conexão local privada preparada, a partir de `expenses-liquibase/`, estes comandos inspecionam a migração:

```bash
liquibase --defaults-file=config/liquibase.local.properties validate
liquibase --defaults-file=config/liquibase.local.properties status --verbose
liquibase --defaults-file=config/liquibase.local.properties update-sql
```

`validate` verifica aspectos do changelog, não prova que o SQL executará nem que a regra está correta. `update-sql` permite revisar o SQL planejado, mas não substitui execução no PostgreSQL. A aplicação real é um passo separado, descrito no [README operacional](../../expenses-liquibase/README.md).

Teste isolado existente:

```bash
python3 tests/validate_schema.py
```

Ele cria um banco temporário, valida/aplica, reaplica sem mudanças, testa integridade, desfaz os oito changesets e reaplica. **Ao adicionar migrações:** revisar contagens fixas, expectativa de tabelas e estratégia de rollback do script; não declarar cobertura de novas sprints sem essa atualização.

Critérios de teste para uma mudança persistente:

- Instalação em banco vazio e atualização a partir da versão anterior com dados.
- Repetição de `update` sem alterações pendentes inesperadas.
- Rejeição de dados inválidos, referências entre casas/competências e violações de unicidade.
- Preservação dos dados válidos e comportamento de campos gerados.
- Rollback e reaplicação quando reversíveis; restauração/correção futura ensaiada quando necessário.
- Compatibilidade com o EF Core e com a versão da API que coexistirá no deploy.
- Concorrência/locks para mudanças que afetem tabelas em uso.

Na futura CI, fixar versões de Liquibase/Java/driver e PostgreSQL, executar em banco descartável e guardar evidências sanitizadas. No deploy, usar uma etapa controlada de migração, evitando que cada réplica da API tente migrar. O pipeline ainda não foi criado.

## 11. Configuração e checklist

Manter segredos em arquivo ignorado ou mecanismo seguro de variáveis do ambiente; não passá-los em exemplos ou em argumentos que exponham credenciais. O `.env` do Compose, User Secrets da API e propriedades JDBC são configurações distintas. URL/usuário do ambiente local não definem automaticamente a configuração de produção.

Registrar execução, versão, resultado e duração conforme o [padrão de observabilidade](OBSERVABILIDADE.md). O processo de migração deve ter `executionId`; SQL com dados e credenciais JDBC não entram na telemetria. O formato precisa ser integrado ao pipeline, sem presumir suporte automático no CLI atual.

Separar a identidade de migração, com DDL necessário, da identidade da aplicação, com permissões de runtime. Preservar as tabelas de controle em seu schema configurado e revisar privilégios dos quatro schemas de negócio.

- [ ] Novo ID estável, ordem explícita e histórico anterior preservado.
- [ ] SQL/rollback no diretório correto, UTF-8 e splitting adequados.
- [ ] PK, FKs, unicidade, nulabilidade, checks e índices revisados.
- [ ] Escopo por casa/competência e histórico financeiro preservados.
- [ ] Locks, transação, volume e compatibilidade de deploy analisados.
- [ ] Preconditions não escondem drift; recuperação de falha definida.
- [ ] Perda de dados no rollback explicitada e alternativa definida quando necessária.
- [ ] Banco vazio, upgrade e integração pertinente testados em base isolada.
- [ ] Modelo físico, mapeamento EF e validações atualizados junto da mudança.
