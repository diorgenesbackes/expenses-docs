# Padrão de testes e Definition of Done

Versão 1.0 — 2026-10-08. Este é o padrão estrutural de qualidade do Expenses. Define o que deve ser planejado, verificado e registrado; sua existência não significa que todos os critérios já estejam implementados ou atendidos.

## 1. Organização e rastreabilidade

A raiz de `testing/` contém somente este documento e as pastas dos testes/ciclos documentados:

```text
testing/
  README.md
  AAAA-MM-DD_HH-mm-ss/
    1-plano.md
    2-execucao.md
```

- Usar data/hora de abertura do ciclo no fuso `America/Sao_Paulo`, com segundos. Registrar também o fuso no plano; se houver colisão, usar o próximo segundo disponível e registrar o instante real no documento.
- Cada pasta identifica um teste ou ciclo documentado, que pode reunir vários casos automatizados. Os casos individuais têm IDs na matriz; não precisam de uma pasta por método unitário.
- Criar os dois arquivos ao abrir o ciclo. `2-execucao.md` começa com status **não executado**; não preencher resultados antecipadamente.
- Registrar nome/objetivo e FDDs relacionadas dentro dos arquivos. O nome da pasta continua apenas data-hora.
- Correções e repetições do mesmo ciclo são registradas como novas tentativas em `2-execucao.md`, preservando a primeira falha. Novo escopo, nova versão candidata ou nova campanha abre outra pasta.
- Planos revisados registram data, alteração e motivo. Não alterar um critério depois da execução para transformar uma falha em aprovação.
- Evidências volumosas/brutas ficam no projeto `expenses-tests/reports/`, fora do Git, com links e IDs no documento de execução. Preservar no Markdown os resultados essenciais, pois artefatos locais podem expirar. Nunca versionar segredos, cookies, tokens ou dados pessoais reais.
- A FDD aponta para cada ciclo em seu `3-testes.md`; o plano e a execução apontam de volta para as FDDs. Isso mantém um único relatório de execução, inclusive quando um ciclo atende várias funcionalidades.

O histórico anterior foi organizado em [2026-10-08_12-02-50](2026-10-08_12-02-50/1-plano.md). Essa data identifica a organização documental; as datas e os IDs das execuções originais foram preservados. A migração não representa novos testes nem aprovação retroativa deste padrão.

## 2. Inventário obrigatório e matriz de cobertura

Manter inventário de **todos os endpoints, todas as telas e todas as jornadas de negócio**. Para cada ciclo, identificar os itens da FDD e as partes do produto afetadas, relacionando-os aos requisitos e à regressão necessária. No aceite completo da FDD, todo item do seu escopo precisa estar implementado e ter seus cenários obrigatórios aprovados. Uma tela ou endpoint ainda inexistente é uma pendência, não um item aprovado ou automaticamente “não aplicável”.

Cada cenário registra:

| Campo | Conteúdo obrigatório |
| --- | --- |
| ID e requisito | Identificador estável e referência ao requisito/FDD. |
| Alvo e camada | Método/rota, tela/jornada ou regra; unidade, integração, API, navegador, carga ou segurança. |
| Preparação | Estado inicial, dados sintéticos, permissões e dependências. |
| Ação | Passos reproduzíveis, entrada e condição de falha quando aplicável. |
| Resultado esperado | Resposta/estado visível e efeitos persistidos; o que não pode ter sido alterado. |
| Resultado real | Aprovado, falhou, não executado, bloqueado, ignorado ou não aplicável. |
| Evidência e pendência | Execução/artefato, divergência, responsável e justificativa de eventual não aplicabilidade. |

Uma chamada HTTP sem verificar resposta/efeitos ou uma tela que apenas abre não comprova a funcionalidade. Contagem de testes e percentual de linhas cobertas não substituem essa matriz. Não há percentual numérico de cobertura de código imposto pelo projeto.

## 3. Cobertura mínima por camada

| Área | O que os testes devem contemplar |
| --- | --- |
| Unitários | Regras, validações, estados, cálculos, limites, erros e cancelamento, com dependências substituídas. Backend/frontend mantêm somente essa camada nos próprios projetos. |
| Todos os endpoints | Método/rota, contrato, campos obrigatórios/opcionais, entradas válidas e inválidas, status, corpo, headers, paginação/limites e efeitos no banco. Conferir sucesso, falha e ausência de efeitos indevidos. |
| Autorização e isolamento | Sem sessão, sessão expirada/revogada, usuário sem permissão e acesso a dados de outro usuário/casa. Nenhum erro pode expor dados indevidos. |
| Todas as telas | Carregamento, conteúdo e ações, formulário válido/inválido, mensagens, estados vazio/loading/erro/sucesso, bloqueio de envio repetido, teclado/foco e navegação. Verificar comportamento acessível e diferentes tamanhos de tela aplicáveis. |
| End-to-end | Jornadas reais de ponta a ponta: abrir telas, preencher, clicar, atravessar a API e confirmar persistência/estado final. Incluir sucesso, falha, nova tentativa e navegação/recarga. Não substituir login visual por um helper num teste que pretende comprovar a tela de login. |
| Banco e migrations | Banco vazio, upgrade com dados sintéticos prévios, integridade, transações, concorrência, idempotência quando prevista, reaplicação e rollback/reaplicação quando suportados. Liquibase é o único responsável pelo schema. |
| Concorrência e repetição | Envio duplo, requisições simultâneas, conflito, replay, limites compartilhados e transições que não podem ser revertidas por requisições atrasadas. |
| Dependências e recuperação | Provedor/banco indisponível, timeout/cancelamento, resposta inválida, falha parcial e recuperação. Incluir reinício e múltiplas instâncias quando houver estado/chaves compartilhadas. |
| Segurança HTTP/navegador | CSRF, origem, cookies, TLS, validação de JWT, limite de corpo, sanitização de erro/logs, cache, headers e comportamento real entre sites/abas conforme o requisito. |
| Performance e carga | Todas as operações com requisito de latência; distribuição por operação, carga nominal, pico e duração aplicáveis, consistência sob carga e recuperação. Registrar limites de CPU/memória e capacidade do gerador. |
| Análise de segurança | Dependências diretas/transitivas, segredos, código/configuração e imagens pertinentes; scan passivo seguido de ativo isolado e autenticado quando aplicável. Relacionar controles ASVS aplicáveis e registrar lacunas. |

Os cenários não aplicáveis exigem justificativa específica. Teste sem necessidade real não deve ser criado apenas para aumentar contagem. Falta de ferramenta, tela ou ambiente é pendência de cobertura, não justificativa de não aplicabilidade.

API via cliente HTTP não abre navegador. Integração em processo não comprova servidor/proxy publicados. DOM simulado não comprova TLS, cookies ou comportamento entre navegadores. Para funcionalidades web, explicitar a matriz de navegadores suportados e o que foi efetivamente executado.

## 4. Ciclo obrigatório do ambiente e banco

Toda suíte que depende de persistência real deve executar o ciclo abaixo em recursos próprios:

1. Identificar a execução, versões/fontes, configuração e dados sintéticos. Recusar conexão com banco de desenvolvimento/produção.
2. Subir PostgreSQL isolado com nome/identificador único e credenciais privadas; aguardar prontidão.
3. Aplicar as migrations com Liquibase e verificar o schema. Health de conexão sozinho não comprova migrations aplicadas. Não usar EF migrations ou `EnsureCreated`.
4. Preparar fixtures controladas. Dados de carga são criados fora da janela medida e identificados no relatório.
5. Subir os componentes necessários ao cenário. Para testes pela rede, usar servidor real, HTTPS e o proxy pertinente; registrar diferenças da topologia final.
6. Executar cenários de sucesso e falha, verificando também consistência e ausência de efeitos proibidos no banco.
7. Coletar resultados e evidências sanitizadas antes da limpeza. Registrar falhas sem repetir silenciosamente até obter sucesso.
8. Encerrar a aplicação/geradores e **destruir o banco, containers/volumes exclusivos, dados e chaves temporárias**. Usar `finally` ou equivalente também em falhas/interrupções tratáveis; nunca fazer limpeza global.
9. Confirmar a remoção e registrar o resultado. Se houver queda forçada que impeça o encerramento, marcar limpeza pendente e fornecer recuperação restrita aos recursos identificados daquela execução.

Unitários não precisam subir banco. Integrações SQL devem usar PostgreSQL real; banco em memória não comprova seu comportamento. Testes externos também usam persistência descartável quando aplicável, mas exigem execução explícita e ambiente externo apropriado. O ciclo local padrão não aciona Cognito/AWS.

## 5. Critérios de desempenho, segurança e evidências

**Desempenho:** definir carga, duração, amostras e limites antes do ensaio. Medir tempo completo percebido pelo cliente, p50/p95/p99, erros, timeouts, taxa pretendida/atingida, iterações não iniciadas e recursos. Separar aquecimento/inicialização fria e não misturar milhares de consultas com poucas amostras de login. Não executar builds/scanners junto da medição. O processo deve retornar falha quando o critério não for atendido.

Para a FDD de autenticação, permanece **p95 < 150 ms por operação**, incluindo dependências no tempo total. Identidade simulada serve à baseline local e não comprova essa meta com Cognito real. Usuários cadastrados, usuários simultâneos e requisições por segundo são medidas diferentes. Ensaio curto não aprova duração prolongada; scripts preparados não equivalem a execuções aprovadas.

**Segurança:** comprovar que o scanner alcançou as rotas protegidas. Restringir destinos ao ambiente autorizado e preservar evidências sem credenciais. Acesso indevido, exposição de segredo ou quebra de integridade são bloqueadores. Achados altos/críticos confirmados bloqueiam publicação. Demais achados precisam de triagem, impacto, ação/responsável e eventual aceite explícito do risco conforme o critério da entrega; não podem ser ignorados para deixar a suíte verde. Falsos positivos devem ter justificativa e exceção restrita e revisável.

**Relatório:** identificar data/hora/fuso, fontes/commits/hashes e alterações locais, versões, ambiente, identidade real/simulada, dados, comandos, resultados por cenário, artefatos, limitações, tentativas anteriores e limpeza. Distinguir “teste aprovado”, “FDD concluída” e “apta para publicação”. Resultado local não homologa a nuvem.

## 6. Definition of Done

Uma FDD só pode ser declarada concluída quando todos os itens aplicáveis estiverem atendidos:

- [ ] Requisitos e critérios de aceite estão relacionados a cenários com IDs, sem lacunas ocultas.
- [ ] Todos os endpoints, telas e jornadas do escopo estão implementados e cobertos; sucesso, falha, limites e regressões pertinentes passaram.
- [ ] Unitários e verificações de build/lint/tipagem pertinentes passaram; integrações e E2E exigidos também passaram na camada adequada.
- [ ] Persistência real foi validada com banco isolado, migrations, dados controlados e limpeza confirmada; upgrade/rollback aplicáveis foram verificados.
- [ ] Autenticação, autorização, isolamento, concorrência e recuperação pertinentes passaram, sem violação de segurança ou integridade.
- [ ] Critérios de performance passaram no ambiente e com as dependências exigidos para o aceite; capacidade, pico e duração aplicáveis têm evidência suficiente.
- [ ] Verificações de segurança foram executadas no escopo definido; não há bloqueadores e riscos residuais têm tratamento/aceite explícito registrado.
- [ ] Não há cenário obrigatório falho, bloqueado, ignorado ou não executado. Itens não aplicáveis estão justificados.
- [ ] Plano e execução estão atualizados, com evidências sanitizadas, pendências e conclusões coerentes com os resultados.
- [ ] `3-testes.md` da FDD aponta para os ciclos e informa a conclusão real; operação/recuperação foram documentadas quando necessárias.

Se parte do escopo foi entregue, registrar **entrega parcial**, com o que passou e o que falta. Uma etapa técnica pode ser concluída sem concluir a FDD inteira. Exceção de escopo não transforma teste não executado em aprovado nem altera requisito de produto sem decisão registrada. Para publicação, também comprovar critérios do ambiente de destino.

## 7. Estrutura dos dois documentos de cada ciclo

### `1-plano.md`

1. **Identificação:** título descritivo, abertura/fuso, responsável, FDDs/requisitos e versão/escopo candidato.
2. **Objetivo e limites:** o que deve ser comprovado, itens fora do escopo e justificativas.
3. **Inventário e matriz:** endpoints, telas, jornadas, IDs dos cenários e resultados esperados por camada, com sucesso/falha.
4. **Ambiente e dados:** topologia, versões, dependências reais/simuladas, migrations, fixtures e isolamento.
5. **Execução e critérios:** comandos, ordem, carga/duração/amostras, critérios de aprovação e interrupção, navegadores aplicáveis.
6. **Evidências e limpeza:** artefatos esperados, sanitização, destruição dos recursos e tratamento de interrupções.
7. **Riscos e revisões:** dependências pendentes, responsáveis e histórico de mudanças no plano.

### `2-execucao.md`

1. **Identificação:** link para o plano, status inicial não executado, responsável e início/fim/fuso efetivos.
2. **Ambiente real:** fontes/versões/configuração/topologia usados, dados, identidade e diferenças em relação ao plano.
3. **Tentativas e resultados:** comandos, IDs de execução, resultado por cenário e totais de aprovados/falhos/ignorados/não executados; preservar repetições e falhas iniciais.
4. **Evidências:** links, métricas e amostras por operação, recursos e limitações; não incluir segredos.
5. **Achados:** defeitos, triagem de segurança, correções, retestes, responsáveis e riscos residuais.
6. **Limpeza:** quais recursos foram destruídos, confirmação e eventuais resíduos/recuperação.
7. **Conclusão:** aceite do ciclo e checklist DoD da FDD, com pendências explícitas. Não aprovar a FDD inteira com base em uma suíte parcial.

Essas seções são o formato obrigatório para novos ciclos. Documentos históricos migrados preservam sua estrutura original e ganham identificação e vínculos; lacunas de evidência continuam registradas como lacunas.
