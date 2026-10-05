# Princípios comuns de desenvolvimento

[Índice dos guidelines](README.md) · [Backend](BACKEND.md) · [Frontend](FRONTEND.md) · [Liquibase](LIQUIBASE.md)

## 1. Arquitetura orientada a responsabilidades

**DEVE:** manter decisões de negócio independentes de HTTP, componentes visuais, detalhes de persistência e fornecedores externos. A direção das dependências de código deve proteger essas regras. Interfaces devem pertencer à camada que precisa da capacidade, e adaptadores externos devem implementá-las. A API pode conhecer a infraestrutura ao montar a aplicação no ponto de composição.

Essa é a aplicação da regra de dependência da [Clean Architecture, de Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html), compatível com a organização descrita pela [Microsoft para aplicações web](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures). O número de pastas ou projetos não comprova o isolamento; o que importa é quem conhece quem e onde estão as regras.

No Expenses:

- Domain expressa conceitos e invariantes; Application coordena casos de uso.
- Infrastructure implementa persistência e integrações; API traduz HTTP e compõe dependências.
- React organiza interação e apresentação; não determina acesso a uma casa nem o resultado definitivo de um rateio.
- Liquibase descreve a evolução persistente do modelo; constraints protegem invariantes mesmo quando a escrita não vem do fluxo habitual.

O SQL não precisa imitar classes e interfaces. O frontend não precisa copiar as quatro camadas do backend. Aplicar separação de responsabilidades conforme as características de cada tecnologia.

## 2. Clean Code como critério de manutenção

| Regra | Aplicação prática |
| --- | --- |
| Nomes comunicam intenção | Preferir `CloseMonthlyPeriod` a `ProcessData`; manter vocabulário de usuário, casa, competência e fechamento consistente. |
| Funções e classes têm responsabilidade coesa | Separar cálculo, persistência e apresentação quando mudam por razões diferentes; não impor um número arbitrário de linhas. |
| Fluxos são explícitos | Guard clauses para entradas inválidas; evitar condicionais profundamente aninhadas e flags booleanas que mudam todo o comportamento. |
| Dados inválidos são difíceis de representar | Validar limites nas fronteiras e invariantes no domínio/banco; usar tipos e estados com significado. |
| Comentários explicam decisões | Registrar por que uma constraint é adiada ou um arredondamento é obrigatório; não narrar cada instrução. |
| Erros mantêm significado | Não engolir exceções, retornar sucesso após falha ou usar `null` para situações indistinguíveis. |
| Abstrações têm consumidores reais | Extrair uma regra compartilhada quando há significado comum, não apenas trechos parecidos. |
| Mudanças são compreensíveis | Não misturar renomeação massiva, troca de biblioteca e alteração de regra em uma única entrega sem necessidade. |

**NÃO DEVE:** manter código morto comentado, credenciais em exemplos, números de domínio sem nome ou um diretório genérico de utilidades acumulando regras de módulos distintos. Convenções de nomes: código e identificadores em inglês, documentação explicativa em português; preservar contratos já publicados.

Simplicidade deve ser avaliada pela mudança atual e pela manutenção previsível. [YAGNI, de Martin Fowler](https://martinfowler.com/bliki/Yagni.html), orienta adiar capacidades especulativas; não justifica omitir segurança, recuperação ou testes necessários ao requisito atual. A [revisão de código do Google](https://google.github.io/eng-practices/review/reviewer/looking-for.html) fornece critérios úteis de desenho, complexidade, testes, nomes e documentação, sem substituir decisões do projeto.

## 3. SOLID aplicado ao contexto

Os princípios orientam limites e contratos; não exigem uma interface para cada classe. A [revisão de SOLID pelo próprio autor](https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html) é a referência conceitual.

| Princípio | Decisão para o Expenses | Sinal de problema |
| --- | --- | --- |
| SRP — responsabilidade única | Controller trata HTTP; caso de uso coordena; política financeira calcula. Um componente visual pode combinar marcação e eventos da mesma interação. | Um serviço calcula rateio, autentica, gera SQL e envia respostas HTTP. |
| OCP — aberto a extensão | Extrair uma política quando existirem variantes reais, como versões de um algoritmo. | Adicionar um framework de plugins para uma única regra. |
| LSP — substituição | Implementações de uma porta mantêm semântica de cancelamento, ausência e erros. | Um repositório alternativo ignora a casa ou lança “não implementado” para operações do contrato. |
| ISP — segregação | Interfaces pequenas, definidas pelo consumidor. | Um leitor de usuários precisa implementar operações de fechamento financeiro. |
| DIP — inversão de dependência | Caso de uso depende de uma porta; EF Core e Cognito ficam em adaptadores. | Regra financeira instancia cliente HTTP ou depende de `HttpContext`. |

No React, composição, funções e hooks podem atender aos mesmos objetivos sem hierarquias de classes. Em migrações, usar coesão e dependências explícitas; não transpor literalmente princípios de objetos para DDL.

## 4. Padrões de projeto: quando usar

| Padrão ou prática | Uso indicado | Limite adotado |
| --- | --- | --- |
| Dependency Injection | Compor serviços e adaptadores, controlar lifetime e substituir dependências em testes. | Não resolver serviços arbitrariamente dentro da regra de negócio. |
| Repository | Expor operações de persistência com significado para o caso de uso. | Evitar repositório genérico que replique todo o EF ou exponha `IQueryable` para HTTP. |
| Unit of Work | Confirmar alterações relacionadas como uma unidade. | `DbContext` já desempenha esse papel; outra abstração precisa demonstrar utilidade. |
| Adapter | Isolar Npgsql, Cognito e o formato de respostas HTTP no frontend. | Não criar camadas que só renomeiam métodos sem proteger um contrato. |
| Strategy | Selecionar algoritmos realmente diferentes e testáveis. | Versionar snapshots antes de permitir que uma estratégia nova afete leituras históricas. |
| Factory | Concentrar criação com invariantes ou seleção complexa. | Um construtor claro basta para objetos simples. |
| Decorator | Aplicar uma política transversal a uma porta bem definida. | Evitar ordem de execução oculta; middleware HTTP serve aos aspectos próprios de HTTP. |
| Composition / custom hook | Reutilizar comportamento de interação no React. | Um hook compartilhado não torna o estado de todas as instâncias global. |
| State / máquina de estados | Tornar transições de fechamento ou tela explícitas quando houver complexidade real. | Um enum e transições validadas podem bastar; biblioteca não é obrigatória. |
| CQRS leve | Separar modelos de leitura e comandos quando suas necessidades divergirem. | Não implica MediatR, event sourcing, outro banco ou mensageria. |

Repository e Unit of Work seguem as definições de [Fowler para Repository](https://martinfowler.com/eaaCatalog/repository.html) e [Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html). As condições de adoção da tabela são decisões deste guideline, não exigências dessas referências.

**NÃO DEVE no MVP:** introduzir event sourcing, outbox, saga distribuída, service mesh ou filas para operações já definidas como síncronas. A integração Cognito/banco requer tratamento explícito de falha e compensação conforme o FDD; isso não transforma automaticamente o sistema em uma arquitetura distribuída de serviços de negócio.

## 5. Contratos, dados e segurança

Cada funcionalidade DEVE responder antes da implementação:

1. Quem pode executá-la e sobre qual recurso/casa?
2. Quais campos, escalas, limites e estados são válidos?
3. Qual é a unidade transacional e o comportamento diante de concorrência?
4. O que acontece se a resposta se perder depois da gravação?
5. Qual comportamento a versão anterior da aplicação terá após a migração?

A autorização deve ser verificada no backend para cada acesso relevante ao recurso. Filtrar a interface ou conhecer um UUID não concede permissão. Aplicar privilégio mínimo e negação por padrão, conforme a [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html).

Contratos devem especificar ausência versus `null`, formato de erro, timezone, paginação, unidade monetária e limites numéricos. O backend valida dados externos, inclusive arquivos e respostas de fornecedores. Logs e testes usam dados sintéticos ou minimizados. Segredos pertencem à configuração privada do ambiente.

O formato obrigatório para novos contratos está no [padrão de APIs síncronas](API-SINCRONA.md). Campos de log, correlação, instrumentos, unidades e critérios de alerta estão no [padrão de observabilidade](OBSERVABILIDADE.md); usá-los na revisão de todas as camadas.

## 6. Registro de decisão

Uma ADR é RECOMENDADA para mudanças que afetem limites de módulos, autenticação, contrato monetário, transações, persistência, implantação ou uma dependência estrutural. Decisões pequenas podem ficar na descrição da mudança.

Modelo proposto para um futuro arquivo `ADR-<numero>-<assunto>.md`:

```markdown
# ADR-<numero>: <decisão>
Status: proposta | aceita | substituída
Data: AAAA-MM-DD

## Contexto e restrições
Problema, requisito e evidências.

## Alternativas
Opções viáveis e custos relevantes.

## Decisão
Escolha e razões.

## Consequências e validação
Limites, compatibilidade, adoção, testes e condição para reconsiderar.
```

O diretório de ADRs ainda não foi criado neste trabalho. O HLD já enumera decisões previstas; reutilizar sua numeração quando forem formalizadas, evitando documentos conflitantes.

## 7. Critérios de revisão e conclusão

Organização de branches, commits e promoções segue o [Git Flow](GIT-FLOW.md). Os critérios abaixo complementam os checks e as evidências exigidos em cada PR.

- [ ] O comportamento atende ao requisito e permanece dentro do HLD vigente.
- [ ] As responsabilidades e dependências estão claras; cada abstração tem um motivo.
- [ ] Contrato, erros, permissões e dados sensíveis foram considerados.
- [ ] Migração, compatibilidade e recuperação foram tratadas quando há mudança persistente.
- [ ] Testes cobrem resultados observáveis, limites e falhas relevantes, sem replicar a implementação.
- [ ] Build, análise estática e testes aplicáveis passaram; testes ignorados foram informados.
- [ ] Documentação e exemplos refletem o comportamento entregue.

Cobertura numérica não substitui cenários. Priorizar casos de alto impacto: isolamento entre casas, soma de centavos, conflitos de versão, expiração de sessão, falha de dependência e preservação do histórico. Não criar uma meta percentual sem conhecer a utilidade e o custo da suíte.
