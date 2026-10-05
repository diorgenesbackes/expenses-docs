### PRD: Contas da Casa MVP Finances

Versão: 1.0
Data: 2026-05-30
Responsável: Diórgenes Backes

---

### Resumo

O Contas da Casa é um novo produto digital para organizar as finanças
compartilhadas de uma família. A aplicação deve centralizar o planejamento
mensal, o acompanhamento de despesas, o controle de pagamentos, o rateio dos
custos entre os responsáveis financeiros e a evolução das rendas
complementares.

O produto deve ser a fonte principal de consulta e atualização das informações
financeiras da casa. Seu propósito é substituir planilhas locais, cálculos
manuais e conferências dispersas por um processo simples, confiável,
compartilhável e atualizado em tempo real.

---

### Contexto e problema

Público-alvo
- Casais e famílias que compartilham despesas e contribuem com rendas diferentes.
- Administrador das finanças da casa, responsável pelo fechamento mensal.
- Co-responsável financeiro, que acompanha custos, pagamentos e sua contribuição.
- Membro consultivo, previsto para evolução futura com permissão limitada.

Cenários de uso chave
- Criar uma casa e convidar o segundo responsável financeiro.
- Abrir o período mensal e registrar despesas, pagamentos, salários e benefícios.
- Calcular automaticamente o rateio proporcional entre os responsáveis.
- Consultar custos, médias, tendências e históricos por categoria.
- Registrar aluguéis e rendas variáveis complementares.
- Fechar o mês com rastreabilidade e consultar fechamentos anteriores.

Onde essa feature será implantada
- O MVP Finances será implantado em um novo sistema desenvolvido do zero.
- A definição detalhada de arquitetura e infraestrutura será realizada em uma
  segunda etapa antes do início do desenvolvimento.

Problemas priorizados
- Alta prioridade: planilhas locais são suscetíveis a erros de preenchimento e
  referência. Já ocorreu um erro de seleção no histórico que afetou os rateios
  seguintes.
- Alta prioridade: o compartilhamento por planilha web ainda gera desencontros,
  espera por atualizações e diferentes formas de realizar apontamentos e
  cálculos.
- Alta prioridade: o modelo atual dificulta visualizar despesas pagas e
  pendentes durante o fechamento.
- Média prioridade: o histórico é difícil de acompanhar e não evidencia
  mudanças no padrão de gastos.
- Média prioridade: a edição e conferência manual aumentam o tempo necessário
  para concluir o fechamento mensal.

---

### Objetivos e métricas

| Objetivo                                                               | Métrica                                                         | Meta                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Reduzir o esforço operacional do fechamento mensal                     | Tempo entre início da edição e confirmação do fechamento        | Até 15 minutos por mês    |
| Eliminar inconsistências no cálculo do rateio                           | Quantidade de erros de cálculo identificados                    | Zero erros                |
| Garantir fechamento sem pagamentos informados em aberto                | Despesas com valor informado marcadas como pagas                | 100 por cento             |
| Estimular a ativação das famílias cadastradas                           | Famílias cadastradas que concluem o primeiro fechamento         | Pelo menos 70 por cento   |
| Estimular o uso recorrente após a ativação                              | Famílias ativadas que concluem o fechamento do mês seguinte     | Pelo menos 80 por cento   |
| Manter colaboração em tempo real dentro da casa                         | Tempo para uma alteração aparecer aos demais usuários conectados | No máximo 500 ms          |

---

### Escopo

Incluso
- Criação de conta de usuário, casa e convite para um segundo responsável
  financeiro.
- Permissões básicas para administrar ou consultar a casa.
- Isolamento seguro dos dados entre casas diferentes.
- Seleção, abertura, fechamento, reabertura e consulta de períodos mensais.
- Histórico de fechamentos e rastreabilidade de alterações relevantes.
- Cadastro, renomeação, ativação e desativação de categorias de despesa.
- Cadastro de despesas recorrentes.
- Registro de um ou mais lançamentos mensais por categoria.
- Entrada rápida de somas, como `100+100+200`.
- Controle de pagamentos pagos e pendentes.
- Observações opcionais por lançamento.
- Indicadores e gráficos consolidados de custos.
- Histórico mensal por categoria de despesa.
- Cadastro de salários líquidos e benefícios compartilhados.
- Cálculo automático do percentual de comprometimento e contribuições em dinheiro.
- Configuração da prioridade de abatimento dos benefícios.
- Cadastro de fontes recorrentes de renda extra.
- Registro de aluguéis e rendas variáveis.
- Indicadores e gráficos históricos de renda extra.
- Migração inicial de dados históricos por arquivo estruturado.
- Exportação dos dados da casa em formato aberto.
- Estratégia de backup e recuperação para produção.
- Interface web responsiva com acessibilidade básica.
- Feedback de sucesso e erro nas ações principais.

Fora de escopo
- Gestão detalhada de despesas de cartão de crédito.
- Importação automática de transações bancárias.
- Integração com bancos, corretoras e plataformas de investimento.
- Regras avançadas de rateio por categoria ou membro.
- Mais de dois responsáveis financeiros no MVP.
- Notificações de contas pendentes e fechamento mensal.
- Planejamento orçamentário e metas de economia.
- Comparação entre orçamento previsto e realizado.
- Categorização automática de lançamentos.
- Anexos e comprovantes de pagamento.
- Aplicativo móvel nativo.
- Relatórios avançados e projeções financeiras.
- Contabilidade empresarial.
- Emissão de documentos fiscais.
- Declaração de imposto de renda.
- Execução de pagamentos ou transferências bancárias.
- Recomendação de investimentos.
- Gestão completa de carteira com cotação de ativos em tempo real.
- Concessão de crédito ou análise de risco financeiro.

---

### Requisitos funcionais

#### RF-01 Criar casa e associar responsáveis financeiros
O sistema deve permitir criar uma casa e associar responsáveis financeiros com
permissões adequadas.

**Fluxo principal**
- O usuário autenticado informa o nome da casa.
- O sistema cria a casa e atribui ao criador o papel de administrador.
- O administrador informa o e-mail do segundo responsável financeiro.
- O sistema envia um convite.
- O convidado acessa ou cria sua conta e aceita o convite.
- O sistema associa o convidado à casa com o papel definido.

**Fluxos alternativos e exceções**
- O administrador pode reenviar ou cancelar um convite pendente.
- Um convidado sem conta pode concluir seu cadastro antes de aceitar o convite.

**Erros previstos**
- Nome da casa não informado.
- E-mail inválido.
- Usuário já associado à casa.
- Tentativa de convite por usuário sem permissão.

**Prioridade:** alta

---

#### RF-02 Isolar dados por casa
O sistema deve manter os dados de cada casa isolados e acessíveis apenas aos
participantes autorizados.

**Fluxo principal**
- O usuário autenticado acessa uma casa da qual participa.
- O sistema valida sua associação e seu papel.
- O sistema libera somente dados e ações autorizados.
- Toda leitura e alteração permanece vinculada à casa selecionada.

**Fluxos alternativos e exceções**
- Um usuário associado a mais de uma casa pode alternar entre elas.
- Um membro consultivo pode visualizar dados sem alterá-los quando esse papel
  estiver disponível.

**Erros previstos**
- Tentativa de acesso a uma casa sem autorização.
- Tentativa de alteração por membro sem permissão.
- Casa inexistente.
- Sessão expirada.

**Prioridade:** alta

---

#### RF-03 Gerenciar períodos mensais
O sistema deve permitir abrir, fechar e consultar períodos mensais, preservando
o histórico de alterações relevantes.

**Fluxo principal**
- O administrador abre um novo período mensal.
- O sistema cria o mês em estado de rascunho.
- Os responsáveis registram despesas, pagamentos, rendas e dados de rateio.
- O sistema valida se todas as despesas com valor informado estão pagas.
- O administrador confirma o fechamento.
- O sistema armazena o resultado consolidado e registra quem fechou o período.

**Fluxos alternativos e exceções**
- Participantes autorizados podem consultar períodos anteriores.
- O administrador pode reabrir um período fechado.
- Ao reabrir, o sistema registra a ação e as alterações posteriores.
- Categorias sem lançamento no mês não geram pendência.

**Erros previstos**
- Fechamento com despesa informada ainda pendente.
- Abertura de período já existente.
- Alteração de período fechado sem reabertura.
- Fechamento ou reabertura por usuário sem permissão.

**Prioridade:** alta

---

#### RF-04 Gerenciar categorias de despesa
O sistema deve permitir cadastrar, renomear, ativar e desativar categorias de
despesa.

**Fluxo principal**
- O administrador acessa a gestão de categorias.
- O administrador cria ou seleciona uma categoria.
- O sistema valida os dados informados.
- O sistema salva a categoria e atualiza os períodos aplicáveis.

**Fluxos alternativos e exceções**
- Uma categoria existente pode ser renomeada.
- Uma categoria pode ser desativada sem apagar seu histórico.
- Uma categoria desativada pode ser reativada.
- Uma categoria pode ser marcada como recorrente.

**Erros previstos**
- Nome obrigatório não informado.
- Categoria duplicada na mesma casa.
- Tentativa de excluir histórico ao desativar uma categoria.
- Alteração por usuário sem permissão.

**Prioridade:** alta

---

#### RF-05 Registrar lançamentos por categoria
O sistema deve permitir registrar um ou mais lançamentos por categoria em cada
mês e calcular seu total automaticamente.

**Fluxo principal**
- O usuário seleciona o período mensal e a categoria.
- O usuário informa um lançamento monetário.
- O sistema valida e salva o lançamento.
- O sistema recalcula o total da categoria e o custo mensal.

**Fluxos alternativos e exceções**
- O usuário pode registrar múltiplos lançamentos na mesma categoria.
- O usuário pode editar ou remover um lançamento antes do fechamento.
- O usuário pode adicionar uma observação opcional ao lançamento.
- Alterações após fechamento exigem reabertura do período.

**Erros previstos**
- Valor inválido.
- Categoria inativa.
- Período fechado.
- Alteração concorrente não reconciliada.

**Prioridade:** alta

---

#### RF-06 Aceitar somas rápidas em campos monetários
O sistema deve aceitar somas rápidas em campos monetários e impedir a execução
de fórmulas arbitrárias.

**Fluxo principal**
- O usuário digita uma expressão como `100+100+200`.
- O sistema valida se a expressão contém somente valores monetários e o operador
  `+`.
- O sistema calcula a soma.
- O campo passa a exibir o resultado final.

**Fluxos alternativos e exceções**
- O usuário pode informar um único valor monetário.
- O usuário pode utilizar centavos.

**Erros previstos**
- Operador não permitido.
- Termo vazio entre operadores.
- Valor não numérico.
- Expressão malformada.

**Prioridade:** média

---

#### RF-07 Controlar pagamentos mensais
O sistema deve permitir marcar despesas mensais como pagas ou pendentes e
recalcular os indicadores correspondentes.

**Fluxo principal**
- O usuário seleciona uma despesa com valor informado.
- O usuário marca a despesa como paga ou pendente.
- O sistema salva o status.
- O sistema atualiza os indicadores do período.

**Fluxos alternativos e exceções**
- Categorias sem lançamento no período não exigem status.
- O usuário pode alterar o status enquanto o período estiver em rascunho.

**Erros previstos**
- Alteração de pagamento em período fechado sem reabertura.
- Tentativa de fechamento com despesa informada pendente.
- Alteração por usuário sem permissão.

**Prioridade:** alta

---

#### RF-08 Exibir indicadores e gráficos de custos
O sistema deve exibir indicadores e gráficos consolidados de custos, incluindo
total mensal, média histórica e maiores despesas.

**Fluxo principal**
- O usuário acessa a visão de custos.
- O usuário seleciona o mês.
- O sistema calcula os indicadores consolidados.
- O sistema exibe os gráficos e valores atualizados.

**Fluxos alternativos e exceções**
- Períodos sem dados exibem estado vazio.
- Alterações em rascunho atualizam os indicadores.

**Erros previstos**
- Falha ao carregar o período.
- Dados inconsistentes impedindo o cálculo.

**Prioridade:** média

---

#### RF-09 Exibir histórico por categoria de despesa
O sistema deve exibir o histórico mensal de uma categoria de despesa
selecionada.

**Fluxo principal**
- O usuário seleciona uma categoria na visão de despesas.
- O sistema recupera seu histórico mensal.
- O sistema exibe um gráfico com a evolução da categoria.

**Fluxos alternativos e exceções**
- Meses sem lançamento aparecem com valor zero.
- Uma categoria sem histórico exibe estado vazio.

**Erros previstos**
- Categoria inexistente.
- Falha ao carregar o histórico.

**Prioridade:** média

---

#### RF-10 Registrar salários líquidos mensais
O sistema deve permitir informar o salário líquido mensal de cada responsável
financeiro.

**Fluxo principal**
- O usuário seleciona o período.
- O usuário informa o salário líquido de cada responsável.
- O sistema valida e salva os valores.
- O sistema recalcula o rateio.

**Fluxos alternativos e exceções**
- Valores podem ser atualizados enquanto o período estiver em rascunho.
- Alterações após fechamento exigem reabertura.

**Erros previstos**
- Valor negativo.
- Valor monetário inválido.
- Responsável financeiro inexistente.
- Período fechado.

**Prioridade:** alta

---

#### RF-11 Gerenciar benefícios compartilhados
O sistema deve permitir cadastrar benefícios compartilhados e configurar sua
prioridade de abatimento.

**Fluxo principal**
- O administrador cadastra um benefício aceito pela casa.
- O administrador define a prioridade de abatimento entre os responsáveis.
- O usuário informa o valor mensal do benefício.
- O sistema salva os dados e recalcula o rateio.

**Fluxos alternativos e exceções**
- O benefício pode ter valor zero em determinado mês.
- A prioridade pode ser alterada para períodos futuros.

**Erros previstos**
- Valor negativo.
- Prioridade incompleta ou inválida.
- Alteração retroativa em período fechado sem reabertura.
- Alteração por usuário sem permissão.

**Prioridade:** alta

---

#### RF-12 Calcular comprometimento e contribuições
O sistema deve calcular automaticamente o percentual de comprometimento e a
contribuição em dinheiro de cada responsável.

**Fluxo principal**
- O sistema recupera o custo mensal, os salários líquidos e os benefícios.
- O sistema calcula o menor percentual inteiro capaz de cobrir o custo.
- O sistema aplica os benefícios conforme a prioridade configurada.
- O sistema exibe contribuições individuais, total comprometido e diferença.

**Fluxos alternativos e exceções**
- Se os dados obrigatórios estiverem incompletos, o sistema sinaliza o rateio
  como pendente.
- Benefícios excedentes seguem a prioridade configurada.

**Erros previstos**
- Dados insuficientes para cálculo.
- Configuração de prioridade inválida.
- Inconsistência entre custo e lançamentos.

**Prioridade:** alta

---

#### RF-13 Impedir contribuições negativas
O cálculo do rateio não deve produzir contribuições em dinheiro negativas.

**Fluxo principal**
- O sistema aplica os abatimentos de benefício na ordem configurada.
- Ao atingir zero para um responsável, o sistema interrompe o abatimento nessa
  parcela.
- O saldo aplicável segue para a próxima parcela conforme a prioridade.
- O sistema exibe somente contribuições iguais ou superiores a zero.

**Fluxos alternativos e exceções**
- Se o benefício superar todas as parcelas aplicáveis, o excedente é
  identificado sem gerar contribuição negativa.

**Erros previstos**
- Resultado negativo detectado durante o cálculo.
- Configuração sem prioridade válida.

**Prioridade:** alta

---

#### RF-14 Armazenar o rateio utilizado no fechamento
O sistema deve armazenar o resultado de rateio utilizado em cada fechamento
mensal.

**Fluxo principal**
- O administrador solicita o fechamento do período.
- O sistema valida os dados do mês.
- O sistema persiste o resultado do rateio com seus valores de origem.
- O sistema associa o resultado ao fechamento.

**Fluxos alternativos e exceções**
- Uma reabertura gera uma nova versão do fechamento após nova confirmação.
- Versões anteriores permanecem consultáveis para auditoria.

**Erros previstos**
- Falha ao persistir o resultado.
- Tentativa de sobrescrever uma versão histórica.
- Rateio pendente ou inconsistente.

**Prioridade:** alta

---

#### RF-15 Gerenciar fontes recorrentes de renda extra
O sistema deve permitir cadastrar fontes recorrentes de renda extra e registrar
seus valores mensais.

**Fluxo principal**
- O usuário acessa a gestão de renda extra.
- O usuário cadastra uma fonte recorrente, como um imóvel alugado.
- O usuário informa seu valor mensal.
- O sistema salva a fonte e recalcula os indicadores.

**Fluxos alternativos e exceções**
- Uma fonte pode ser renomeada, desativada ou reativada sem perder histórico.
- O valor pode variar entre períodos.

**Erros previstos**
- Nome não informado.
- Valor inválido.
- Fonte duplicada.
- Alteração por usuário sem permissão.

**Prioridade:** alta

---

#### RF-16 Sugerir valores recorrentes no novo mês
Ao abrir um novo mês, o sistema deve sugerir o último valor conhecido para
fontes recorrentes de renda.

**Fluxo principal**
- O administrador abre um novo período.
- O sistema localiza o último valor conhecido de cada fonte recorrente ativa.
- O sistema preenche os valores como sugestões.
- O usuário revisa e confirma ou altera os valores.

**Fluxos alternativos e exceções**
- Fontes sem histórico são iniciadas sem valor.
- Uma sugestão pode ser substituída manualmente.

**Erros previstos**
- Falha ao recuperar o último valor.
- Fonte recorrente inativa incluída indevidamente.

**Prioridade:** média

---

#### RF-17 Registrar rendas variáveis
O sistema deve permitir registrar rendas variáveis mensais e consolidar o total
de renda extra.

**Fluxo principal**
- O usuário seleciona o período.
- O usuário registra um ou mais valores de renda variável.
- O sistema valida e salva os lançamentos.
- O sistema recalcula a renda extra mensal.

**Fluxos alternativos e exceções**
- O usuário pode editar ou remover lançamentos enquanto o período estiver em
  rascunho.
- O sistema aceita somas rápidas nos campos monetários aplicáveis.

**Erros previstos**
- Valor inválido.
- Período fechado sem reabertura.
- Alteração por usuário sem permissão.

**Prioridade:** alta

---

#### RF-18 Exibir indicadores e gráficos de renda extra
O sistema deve exibir indicadores e gráficos históricos de renda extra.

**Fluxo principal**
- O usuário acessa a visão de renda extra.
- O usuário seleciona o período.
- O sistema consolida fontes recorrentes e rendas variáveis.
- O sistema exibe total, médias e evolução histórica.

**Fluxos alternativos e exceções**
- Períodos sem renda extra exibem valor zero e estado apropriado.
- Alterações em rascunho atualizam os indicadores.

**Erros previstos**
- Falha ao carregar o período.
- Dados inconsistentes impedindo a consolidação.

**Prioridade:** média

---

#### RF-19 Migrar dados históricos
O sistema deve permitir migrar dados históricos por arquivo estruturado.

**Fluxo principal**
- O administrador seleciona um arquivo compatível.
- O sistema valida o formato e os registros.
- O sistema apresenta um resumo da migração.
- O administrador confirma a operação.
- O sistema importa os dados e registra o resultado.

**Fluxos alternativos e exceções**
- Registros inválidos são informados antes da confirmação.
- A migração pode ser cancelada antes da gravação.

**Erros previstos**
- Arquivo incompatível.
- Campos obrigatórios ausentes.
- Duplicidade de períodos ou categorias não reconciliada.
- Falha parcial durante a importação.

**Prioridade:** média

---

#### RF-20 Exportar dados da casa
O sistema deve permitir exportar os dados da casa em formato aberto.

**Fluxo principal**
- O administrador solicita a exportação.
- O sistema valida sua permissão.
- O sistema gera o arquivo com os dados da casa.
- O sistema disponibiliza o arquivo para download.

**Fluxos alternativos e exceções**
- O formato aberto definitivo será definido na etapa técnica.
- A exportação pode ser gerada de forma assíncrona caso o volume exija.

**Erros previstos**
- Usuário sem permissão.
- Falha na geração do arquivo.
- Exportação temporariamente indisponível.

**Prioridade:** média

---

### Requisitos não funcionais

Performance
- Operações síncronas frequentes, como consultar um mês ou salvar um
  lançamento, devem apresentar latência `p95` menor que `150 ms`.
- Alterações devem aparecer para os demais usuários conectados à mesma casa em
  até `500 ms`.

Disponibilidade
- O sistema deve apresentar disponibilidade mensal mínima de `99,9%` em
  produção.

Segurança e autorização
- Autenticação deve ser obrigatória.
- A autorização deve considerar papel e casa.
- Dados financeiros devem ser criptografados em trânsito e em repouso.
- A gestão de sessão deve seguir práticas seguras.
- Alterações sensíveis devem gerar trilha de auditoria.

Observabilidade
- O sistema deve produzir logs estruturados.
- O sistema deve coletar métricas de erro por endpoint.
- O sistema deve oferecer tracing distribuído ponta a ponta.
- Logs não devem registrar valores financeiros sensíveis.

Confiabilidade e integridade de dados
- Operações de gravação devem ser idempotentes quando aplicável.
- O sistema não pode gerar duplicidade silenciosa de lançamentos.
- Cálculos financeiros devem utilizar precisão decimal adequada para valores
  monetários.
- Alterações em períodos fechados devem ser rastreáveis.
- O produto deve contar com estratégia testada de backup e recuperação.

Compatibilidade e portabilidade
- O frontend deve ser responsivo e utilizável em navegadores modernos de
  desktop e dispositivos móveis.
- A API deve utilizar REST com JSON e versionamento explícito.
- A estratégia de empacotamento da aplicação será definida na etapa técnica.

Compliance
- O tratamento de dados pessoais deve atender à LGPD.

Acessibilidade no frontend consumidor
- A interface deve oferecer navegação por teclado.
- Campos e controles devem possuir rótulos adequados.
- O frontend deve utilizar contraste apropriado.
- As ações mais frequentes devem apresentar feedback visual imediato.

---

### Arquitetura e abordagem

Abordagem
- A arquitetura será definida em uma segunda etapa dedicada, antes do início do
  desenvolvimento.
- A solução deverá suportar um novo sistema colaborativo, atualização em tempo
  real, isolamento por casa, segurança de dados financeiros e os requisitos não
  funcionais deste documento.

Componentes
- Componentes pendentes de definição na etapa de arquitetura.

Integrações
- Integrações pendentes de definição na etapa de arquitetura.
- Integrações com bancos, corretoras e plataformas de investimento estão fora
  do escopo do MVP.

### Decisões e trade-offs

#### Decisão: definir arquitetura e infraestrutura em uma segunda etapa dedicada
- **Justificativa:** ainda não existem definições técnicas aprovadas e a
  arquitetura deve ser escolhida considerando segurança, colaboração em tempo
  real, disponibilidade e evolução futura.
- **Trade-off:** o desenvolvimento do MVP não pode começar antes da aprovação
  dessas definições.

#### Decisão: utilizar frontend web responsivo e API REST JSON versionada
- **Justificativa:** essa abordagem atende ao acesso por navegadores modernos e
  estabelece um contrato explícito para o frontend.
- **Trade-off:** detalhes de implementação, empacotamento e infraestrutura
  permanecem pendentes para a etapa técnica.

---

### Dependências

#### Organizacional: design do frontend
A implementação depende da entrega do design da interface web responsiva,
incluindo fluxos principais, estados vazios, erros, feedback de ações e
diretrizes de acessibilidade.

#### Técnica: definição de arquitetura
Uma etapa técnica deve definir a arquitetura do sistema, os componentes, os
padrões de comunicação, a persistência, a estratégia de atualização em tempo
real e a abordagem de backup antes do desenvolvimento.

#### Técnica: definição de infraestrutura
A infraestrutura de desenvolvimento, homologação e produção deve ser definida
antes do início da implementação, incluindo observabilidade, segurança,
disponibilidade e estratégia de recuperação.

---

### Riscos e mitigação

#### A definição de arquitetura e infraestrutura pode atrasar o início do desenvolvimento
- **Probabilidade:** baixa
- **Impacto:** atraso no início da implementação do MVP.
- **Mitigação:**
  - Realizar uma etapa técnica dedicada para definir arquitetura e
    infraestrutura antes do início do desenvolvimento.
  - Aprovar as decisões necessárias antes de iniciar a implementação.
- **Plano de contingência:** adiar o início da implementação até a aprovação das
  definições técnicas.

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Uma conta autenticada consegue criar uma casa e convidar o segundo
  responsável financeiro.
- Um usuário não consegue acessar dados de uma casa da qual não participa.
- Um usuário com permissão somente de consulta não consegue alterar dados.
- Um administrador consegue abrir, fechar, consultar e reabrir um período
  mensal.
- O sistema bloqueia o fechamento quando existe despesa com valor informado e
  status pendente.
- Categorias sem lançamento no período não impedem o fechamento.
- Um usuário autorizado consegue cadastrar, renomear, ativar e desativar
  categorias sem apagar o histórico.
- O total mensal de uma categoria corresponde à soma de seus lançamentos.
- O custo mensal da casa corresponde à soma das categorias ativas no período.
- A entrada `100+100+200` é aceita como soma rápida e resulta em `500`.
- Expressões monetárias com operadores não permitidos são rejeitadas.
- O status pago ou pendente atualiza os indicadores do mês.
- A visão de custos exibe total mensal, média histórica e maiores despesas.
- O histórico de uma categoria exibe sua evolução mensal e considera zero nos
  meses sem lançamento.
- O sistema registra os salários líquidos dos dois responsáveis no mês.
- O sistema cadastra benefícios e aplica a prioridade de abatimento configurada.
- O rateio calcula o menor percentual inteiro capaz de cobrir o custo mensal.
- Nenhuma contribuição individual em dinheiro pode ser inferior a zero.
- O fechamento preserva o resultado do rateio e os valores utilizados no
  cálculo.
- A reabertura de um período fechado gera registro rastreável das alterações.
- O sistema permite cadastrar fontes recorrentes de renda extra.
- Um novo mês sugere o último valor conhecido para fontes recorrentes ativas.
- O sistema consolida fontes recorrentes e lançamentos variáveis na renda extra
  mensal.
- A visão de renda extra exibe indicadores e histórico.
- O sistema valida e importa um arquivo estruturado de histórico após
  confirmação do administrador.
- O administrador consegue exportar os dados da casa em formato aberto.
- Alterações aparecem para os demais usuários conectados à mesma casa em até
  `500 ms`.
- Operações síncronas frequentes apresentam latência `p95` menor que `150 ms`.
- O sistema atende à disponibilidade mensal mínima de `99,9%` em produção.
- Logs e traces não registram valores financeiros sensíveis.
- O tratamento de dados pessoais atende à LGPD.

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários com cobertura mínima de `90%` do código.
- Testes de integração para todas as APIs.
- Testes de segurança para frontend e APIs.
- Testes automatizados de tela com roteiro que cubra todas as regras de
  negócio e substitua o teste manual.
- Testes de carga para validar latência, disponibilidade operacional e
  atualização em tempo real.
- Testes de estresse para avaliar o comportamento do sistema acima da carga
  esperada.

Estratégia de validação
- O deploy em `dev` deve ser bloqueado quando houver falha nos testes unitários
  ou de integração.
- O deploy em `hlg` deve ser bloqueado quando houver falha nos testes
  automatizados de tela, carga ou estresse.
- O deploy em produção não deve possuir bloqueio automático associado à
  execução desses testes.
- A suíte automatizada deve validar as regras de negócio, a segregação por
  casa, as permissões, os cálculos financeiros e os requisitos mensuráveis de
  performance.
