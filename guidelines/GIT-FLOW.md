# Git Flow do Expenses

[Índice](README.md) · [Princípios comuns](PRINCIPIOS.md) · [Liquibase](LIQUIBASE.md)

Base: [Markdown fornecido pelo usuário](referencias/git-flow-base.md), preservado como referência. Este guia mantém `main`, `hlg`, `dev`, `feature/`, `task/`, `merge/` e `fix/`, complementando revisão, integração, releases e tratamento de banco.

**Status:** processo definido para adoção. Na análise de 2026-10-05, esta pasta não estava inicializada como repositório Git. Nenhuma branch, proteção, integração com provedor, tag ou pipeline foi criada por este documento. Comandos abaixo são exemplos para quando o remoto `origin` e as branches existirem.

## 1. Modelo adotado

O Expenses usa **promoção seletiva por história**. Cada história nasce de `main` e é integrada separadamente em `dev`, `hlg` e `main`. Passar por um ambiente é uma etapa de validação, não autorização para incorporar todo o conteúdo da branch daquele ambiente.

```text
main → feature/123 → task/456
                    task/457

task/456 e task/457 → feature/123

feature/123 → dev    (integração)
feature/123 → hlg    (homologação)
feature/123 → main   (entrega aprovada)

main → fix/789 → main
       fix/789 → hlg e dev
```

As setas iniciais representam criação; as demais, PRs de integração. O diagrama representa o processo, não um histórico de commits com SHAs idênticos entre ambientes. Não usamos uma branch `develop` nem uma família `release/*` adicional neste modelo.

## 2. Branches e nomenclatura

| Branch | Criada a partir de | Finalidade / destino |
| --- | --- | --- |
| `main` | Base estável inicial | Mudanças prontas para produção; recebe histórias homologadas e hotfixes. |
| `hlg` | Base estável inicial | Conjunto selecionado de histórias em homologação. |
| `dev` | Base estável inicial | Integração das histórias prontas para teste conjunto. |
| `feature/<id-historia>` | `main` atualizada | História completa; promovida para `dev`, depois `hlg` e finalmente `main`. |
| `task/<id-subtask>` | Respectiva `feature/*` | Subtarefa; PR exclusivamente para sua história. |
| `merge/<origem>-into-<destino>` | Branch de destino | Resolução de conflito de uma integração específica; PR somente para esse destino. |
| `fix/<id-tarefa>` | `main` alinhada à produção | Correção emergencial; PRs para `main`, `hlg` e `dev`. |

Exemplos: `feature/123`, `task/456`, `merge/feature-123-into-dev`, `fix/789`. Na branch de resolução, substituir `/` dos nomes de origem/destino por `-`. IDs devem ser únicos e vinculados à história/tarefa; a descrição detalhada fica no tracker e no PR. Não reutilizar um ID de branch concluída para uma demanda diferente.

Uma história pequena pode ser desenvolvida diretamente em sua `feature/*`, com revisão na promoção. Usar `task/*` quando houver subtarefas ou colaboração; não criar subtarefa vazia apenas para cumprir a estrutura. Melhorias de documentação, manutenção e refatoração planejadas seguem o mesmo fluxo de histórias.

## 3. Integrações permitidas e restrições

| Origem → destino | Condição |
| --- | --- |
| `task/*` → sua `feature/*` | Revisão da subtarefa e checks pertinentes. |
| `feature/*` → `dev` | História pronta para integração. |
| `feature/*` → `hlg` | Mesmo conteúdo da história validado em dev; dependências declaradas. |
| `feature/*` → `main` | Homologação aprovada e candidato de produção validado. |
| `fix/*` → `main`, `hlg`, `dev` | Correção emergencial testada e propagada, sem aguardar histórias não relacionadas. |
| `main` → `dev`, `hlg` | Sincronização controlada após publicação, por PR. |
| `main` → `feature/*` ativa | Atualizar a história com a base estável; revisar impacto e repetir validações. |
| `merge/*` → seu destino | Somente a integração indicada no nome e no PR. |

**Não integrar:** `dev` → `hlg`, `hlg` → `main`, `dev` → `main`, ou `dev`/`hlg` → `feature/*`/`fix/*`. Esses caminhos podem carregar histórias ainda não aprovadas. `task/*` não é promovida diretamente a um ambiente.

Sincronizar a produção de volta aos ambientes é uma complementação ao documento-base: mantém a ancestralidade e as correções estáveis sem promover histórias que só existem em dev/hlg. Se houver conflito nessa sincronização, usar uma branch `merge/main-into-<destino>` e validar seu resultado.

## 4. Passo a passo de uma história

### 4.1. Criar história e subtarefas

Com alterações locais preservadas e diretório de trabalho limpo:

```bash
git status
git fetch origin
git switch -c feature/123 origin/main
git push -u origin feature/123
```

Para uma subtarefa, depois que a feature estiver no remoto:

```bash
git fetch origin
git switch -c task/456 origin/feature/123
git push -u origin task/456
```

Trabalhar em commits pequenos e coesos. Abrir PR `task/456` → `feature/123`, explicando resultado e validação. Se outra subtarefa mudar o contrato necessário, atualizar a task a partir de sua feature e resolver o impacto antes da revisão final.

### 4.2. Integrar em dev

Após reunir as subtarefas, abrir `feature/123` → `dev`. Testar o resultado da integração com as outras histórias. Registrar o SHA da feature, o commit resultante da integração e as evidências. Não considerar o build isolado da task suficiente para comprovar integração.

### 4.3. Homologar

Abrir **a mesma feature** → `hlg`. Se surgir bug, criar uma task a partir da feature ou corrigir nela quando a demanda for pequena; então promover o novo conteúdo novamente a `dev` e `hlg`. Não corrigir código diretamente em `hlg`.

A aprovação de QA fica vinculada ao SHA aprovado e aos artefatos testados. Qualquer commit posterior que mude o conteúdo invalida a aprovação anterior e exige nova validação do impacto. Uma tarefa existente em dev/hlg não deve ser reescrita por rebase/force push para aparentar uma promoção limpa.

### 4.4. Preparar produção

Antes de integrar `feature/123` → `main`:

1. Atualizar a história com mudanças estáveis de `main`, quando houver, e repetir as validações afetadas.
2. Conferir que o PR contém apenas a história aprovada e dependências já publicadas.
3. Testar o candidato formado por **main atual + história aprovada**.
4. Registrar artefatos, migrations, configuração necessária e estratégia de recuperação.
5. Integrar por PR após os checks e aprovações; publicar conforme a seção de releases.

**Homologação agregada não basta para comprovar o candidato:** `hlg` pode conter outra história ainda não aprovada. Uma aprovação em `hlg` precisa ser complementada por validação isolada do candidato de produção quando os conjuntos diferirem. Usar ambiente/base isolados para esse ensaio; se isso ainda não estiver disponível, não declarar que um artefato diferente foi homologado.

Se `main` mudar depois desse ensaio, reconstruir/revalidar o candidato antes de publicar. Não substituir o conteúdo homologado apenas porque a branch ainda tem o mesmo nome.

## 5. Dependências entre histórias

Se B depende de A, registrar a dependência nos dois PRs. B só pode ser promovida a um destino depois que A estiver nele. Para produção, A precisa estar publicada antes, ou A+B devem ser tratadas como uma entrega conjunta com escopo e candidato explicitamente homologados.

Não resolver dependência mesclando `dev` inteira em B. Preferir publicar primeiro o contrato compatível de A e atualizar B com `main`. Se não for possível entregar separadamente, reorganizar as subtarefas sob uma história que represente a entrega coesa; não ocultar a dependência no histórico.

Essa regra vale para componentes React, contratos da API, tabelas/FKs e configurações de infraestrutura. Um banco de dev com tabelas de A pode fazer B passar indevidamente quando a produção ainda não tem A.

## 6. Estratégia de merge e conflitos

**Padrão:** merge commit, equivalente a `--no-ff`, nas integrações por PR. Preservar a ancestralidade da feature/fix ao integrá-la em mais de uma branch. Não usar squash ou rebase merge nessas promoções. Adotar merge commit também para tasks simplifica a política inicial do repositório.

Merge commits preservam os commits de origem; squash não registra essa relação como merge. Essa diferença fundamenta a escolha neste fluxo de múltiplos destinos. Referência: [Git — merge](https://git-scm.com/docs/git-merge).

Não fazer rebase/force push em `main`, `hlg`, `dev` ou em branches compartilhadas/promovidas. Não habilitar exigência de histórico linear nas branches permanentes se ela impedir os merge commits escolhidos. A UI de atualização de branch também precisa respeitar as restrições de origem: “atualizar feature com dev” não é um atalho permitido.

### Resolver conflito de integração

Uma `merge/*` nasce do **destino**, incorpora a origem e volta somente para aquele destino. Exemplo de preparação para dev:

```bash
git fetch origin
git switch -c merge/feature-123-into-dev origin/dev
git merge --no-ff --no-commit origin/feature/123
```

Resolver os arquivos conscientemente, revisar o diff, executar os checks, adicionar apenas arquivos resolvidos, criar o commit e abrir PR da branch intermediária para `dev`. Para desistir enquanto o merge está em andamento, usar `git merge --abort`; iniciar com a árvore limpa evita misturar trabalho anterior à resolução.

**Nunca promover essa branch intermediária para hlg/main:** ela contém todo o destino original. Se a resolução revelar uma correção de negócio, fazer a correção também na feature canônica, repetir seus testes e promover a nova revisão; não deixar a única implementação correta escondida numa resolução de ambiente.

Conflito com `main` deve preferencialmente ser resolvido ao atualizar a feature a partir de `main`, antes da homologação final. Uma branch `merge/*` para produção é exceção para a resolução revisada, com candidato novamente validado; não dispensa esse ciclo.

## 7. Pull requests e proteções

Cada PR DEVE informar origem/destino, história, resultado, validação, dependências e impacto persistente. Uma promoção referencia os PRs anteriores e o SHA aprovado. Evitar PRs que misturem reorganização ampla, dependência nova e comportamento sem relação.

Modelo de descrição:

```markdown
## Mudança
Problema, comportamento resultante e ID da história/tarefa.

## Integração
Origem → destino; SHA da origem; PRs/evidências anteriores; dependências.

## Validação
Checks executados, resultado, testes ignorados e justificativa aplicável.

## Banco, contrato e publicação
Changesets, compatibilidade, configuração, artefatos e recuperação.
Usar “não se aplica” apenas quando realmente não houver impacto.
```

Política proposta para `main`, `hlg` e `dev`:

- Alterações somente por PR, sem push direto, force push ou exclusão.
- Pelo menos uma revisão de outra pessoa, checks aplicáveis aprovados e conversas resolvidas.
- Nova alteração de conteúdo exige nova revisão; promoção a `hlg` exige evidência de dev e a `main` exige aceite de homologação.
- Serializar integrações que alterem o candidato; validar contra o destino atualizado, sem trazer todo esse destino para a feature.
- Restringir bypass administrativo e publicação/tag a responsáveis definidos.

As proteções dependem do provedor e do plano. [GitHub — protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) é uma referência de implementação, não uma escolha já feita de hospedagem. Checks de promoção/origem exigem automação específica; não presumir que exigir PR valide sozinho todas as regras.

Enquanto o projeto tiver um único mantenedor, registrar a revisão própria como exceção visível; ela não equivale à aprovação independente. Não configurar uma exigência impossível de satisfazer para depois contorná-la rotineiramente.

## 8. Commits

Adotar [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) nos commits de trabalho e nos títulos de PR: `<tipo>(<escopo>): <descrição>`. Escopos sugeridos: `api`, `application`, `domain`, `infrastructure`, `ui`, `liquibase`, `infra`, `docs`, `git`.

`infrastructure` identifica a camada .NET; `infra` identifica Compose, containers e configuração de implantação.

```text
feat(api): adicionar consulta de competência
fix(ui): preservar os campos após falha de validação
docs(git): definir promoção seletiva de histórias
test(liquibase): verificar isolamento entre casas
refactor(application): separar política de rateio
```

Tipos iniciais: `feat`, `fix`, `docs`, `test`, `refactor`, `perf`, `build`, `ci`, `chore`, `revert`. Usar `!`/`BREAKING CHANGE:` quando houver incompatibilidade real. Incluir referência à história no corpo/rodapé ou associação do PR, sem colocar segredos no texto.

`fix:` como tipo de commit significa correção; não obriga usar branch `fix/*`. Correções normais/bugs de homologação permanecem na feature; `fix/*` é reservada ao hotfix de produção. Commits de merge gerados pelo provedor podem manter seu formato próprio; a descrição do PR registra a intenção.

## 9. Checks por área

Executar checks sobre a revisão avaliada e também sobre o resultado integrado quando houver impacto. O padrão inicial é:

| Área alterada | Validação |
| --- | --- |
| Backend | Em `expenses-service/`: `dotnet build Expenses.sln`, `dotnet test Expenses.sln --no-build`; PostgreSQL real via `python3 scripts/test-postgres.py` quando houver impacto no acesso/integridade. |
| Frontend | Em `expenses-ui/`: `npm ci`, `npm run lint`, `npm run build`; testes funcionais quando a suíte for adotada. |
| Liquibase | Em `expenses-liquibase/`: `python3 tests/validate_schema.py`, atualizado para a entrega; verificar instalação limpa e upgrade com dados. |
| Infraestrutura | Validar Compose e build dos componentes afetados conforme seu README; conferir portas, volumes e health sem alterar dados compartilhados. |
| Documentação | Links, exemplos e consistência com código/contratos. |

Mudança transversal executa a combinação aplicável. Filtros por diretório na futura CI não podem ignorar que SQL afeta EF/API e que um contrato da API afeta UI. Teste de integração ignorado não conta como teste aprovado. Não criar um `npm test` obrigatório enquanto esse script não existir.

Segredos, `.env`, propriedades locais do Liquibase, `bin/`, `obj/`, `node_modules/` e artefatos temporários não entram nos commits. Revisar os arquivos staged, em vez de adicionar indiscriminadamente todo o diretório.

## 10. Releases e artefatos

**main é a linha estável; a versão efetivamente em produção é identificada pela tag e pelo registro de deployment.** Merge não comprova implantação. Serializar publicações e evitar acumular em main histórias aprovadas que não fazem parte da próxima entrega.

Adotar uma tag de entrega do monorepo, `vMAJOR.MINOR.PATCH`, anotada e imutável. A política de incremento segue [Semantic Versioning](https://semver.org/spec/v2.0.0.html): incompatibilidade, funcionalidade compatível e correção compatível. Versão da entrega não é a versão `/v1` do contrato HTTP nem a versão de um pacote de terceiro. Escolher a primeira versão ao publicar; este guia não cria uma tag inicial.

Cada entrega registra:

- Tag e SHA de main, SHA/conteúdo do candidato homologado e PRs incluídos.
- Digests das imagens API/UI, versão dos manifests/configuração pública e pacote/checksums das migrações.
- Changesets novos, ordem de execução, compatibilidade com a aplicação anterior e evidências de upgrade.
- Responsável, instante, ambiente, verificação pós-deploy e resultado efetivo.

Construir uma vez o candidato e promover os mesmos artefatos imutáveis quando o conteúdo final for equivalente. Um merge commit pode mudar o SHA sem mudar a árvore; conferir equivalência de conteúdo e registrar a proveniência. Se houver resolução de conflito, alteração de dependência, configuração embutida no frontend ou rebuild diferente, validar novamente o artefato antes de publicá-lo.

Após sucesso, abrir PRs de sincronização `main` → `hlg` e `main` → `dev`. Se a publicação falhar, registrar o resultado e executar a estratégia de recuperação; não mover uma tag existente para outro commit nem afirmar que main representa o que está rodando.

## 11. Hotfix em produção

1. Confirmar a tag/artefato em produção e o alinhamento de main. Se main estiver à frente por uma publicação incompleta, reconciliar o escopo antes de seguir; não incluir mudanças não publicadas no hotfix por acidente.
2. Criar `fix/789` a partir da base main estável e limitar a mudança ao incidente.
3. Executar testes direcionados e checks aplicáveis; revisar o PR `fix/789` → `main`.
4. Publicar nova versão patch compatível e verificar o incidente em produção.
5. Abrir PRs da **mesma fix** para `hlg` e `dev`; resolver conflitos com branches intermediárias próprias de cada destino.
6. Sincronizar main nos ambientes e features ativas que precisam da correção, evitando reintroduzir o bug.

Urgência encurta escopo e prazo de revisão, não elimina rastreabilidade. Não aguardar a aprovação de outra história para levar a fix a produção. Remover a branch somente após a propagação e o registro da entrega.

## 12. Liquibase em branches paralelas

Changelogs e SQL pertencem à mesma história que altera o comportamento. Aplicar as regras do [guia Liquibase](LIQUIBASE.md), em especial:

- Reservar IDs/caminhos antes de aplicar em ambiente compartilhado, evitando duas histórias com o mesmo arquivo/changeset.
- Registrar includes em ordem explícita. Conflito no master exige análise da dependência, não apenas manter as duas linhas em qualquer ordem.
- Não editar, renomear ou remover da história um changeset já aplicado em dev/hlg compartilhado para corrigir seu efeito. Criar correção incremental; não limpar checksums como parte do merge.
- Homologar a promoção a partir do estado real de produção, além de uma base vazia. Tabelas de outra história em hlg não podem mascarar dependência ausente.
- Atualizar contagens e rollback dos validadores: o script atual contém expectativas fixas dos oito changesets e 33 tabelas da sprint-1.

Retirar uma história do código **não desfaz o banco**. Não resetar dev/hlg para main, eliminar volumes ou aplicar rollback destrutivo para “limpar o ambiente” sem um procedimento específico. Mudanças compatíveis podem permanecer fisicamente sem uso até a correção/entrega seguinte; documentar a divergência. O schema necessário a um candidato precisa ser conhecido e reproduzível.

## 13. Reversão e encerramento

Reverter código por novo commit/PR, preservando o histórico. Em merge commits, a escolha do parent de `git revert` altera o resultado e precisa ser revisada. Reverter um merge não apaga sua ancestralidade: tentar integrar a mesma feature novamente pode não restaurar as mudanças. Planejar correção futura ou reversão da reversão com testes. Referência: [Git — revert](https://git-scm.com/docs/git-revert).

Rollback de imagem só é viável se a versão anterior continuar compatível com o schema/configuração atuais. Dados financeiros e snapshots não são recuperados por Git. Para incidentes, privilegiar correção incremental e a estratégia de recuperação documentada na entrega.

Apagar tasks após integração na feature e encerrar a feature após publicação/sincronização, mantendo histórico nos PRs/tags. Não habilitar exclusão automática da feature ao primeiro merge em dev: ela ainda será promovida a outros destinos. Branches `merge/*` terminam após sua integração específica; manter registro de origem, destino e revisão.

## 14. Checklist de adoção

- [ ] Inicializar/conectar o repositório e estabelecer a base estável, em tarefa própria.
- [ ] Criar main/hlg/dev e definir responsáveis e vínculo com ambientes.
- [ ] Configurar PRs, merge commits e proteções, sem histórico linear incompatível.
- [ ] Preservar branches multi-destino até o término das promoções.
- [ ] Criar checks de origem/destino, validação por área e resultado integrado.
- [ ] Disponibilizar homologação do candidato e banco isolado para migrations.
- [ ] Configurar registro de artefatos, tags imutáveis, deployments e recuperação.
- [ ] Revisar toda promoção por SHA/conteúdo, dependências e evidências.

Esses itens descrevem a implantação futura do processo, não configurações já habilitadas.
