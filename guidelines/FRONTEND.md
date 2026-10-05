# Guideline de frontend

[Índice](README.md) · [Princípios comuns](PRINCIPIOS.md) · [Execução do frontend](../../expenses-ui/README.md)

Aplicação: React 19, TypeScript 6, Vite 8 e Oxlint. O frontend atual ainda é o template inicial, sem autenticação, roteamento de negócio, biblioteca de formulários, cache de consultas ou suíte de testes. As estruturas e ferramentas sugeridas abaixo são propostas de adoção incremental.

## 1. Organização por funcionalidade

**RECOMENDADO:** manter componentes, hooks, contratos e testes próximos da funcionalidade que os utiliza. Extrair para compartilhamento quando existir uso comum real. Estrutura ilustrativa para quando surgirem funcionalidades:

```text
src/
  app/                        # inicialização, rotas e providers
  features/
    identity/
      api/                    # contratos e chamadas desse módulo
      components/
      hooks/
    finance/
      api/
      components/
      hooks/
      model/                  # funções puras e tipos da interface
  shared/
    api/                      # transporte HTTP e interpretação de erros
    ui/                       # controles reutilizados
    lib/                      # utilidades pequenas com finalidade clara
```

Criar apenas as pastas que tiverem conteúdo. `shared` não deve depender de `features`; funcionalidades não devem importar detalhes internos umas das outras. Preferir contratos públicos pequenos quando precisarem colaborar. Evitar arquivos de reexportação que criem dependências circulares ou escondam o custo de importação.

Componentes visuais não devem conhecer SQL, Cognito ou o schema físico. O adaptador HTTP converte o contrato da API no modelo necessário à tela. Isso aplica os limites da Clean Architecture sem reproduzir camadas e classes do backend dentro do React.

## 2. Componentes, composição e hooks

- **DEVE:** manter renderização pura, sem alterar props/estado recebido nem executar gravações durante render.
- Atualizar arrays e objetos de estado por novas referências; preferir composição a herança.
- `useState`, `useEffect` e os hooks convencionais devem ser chamados no topo do componente/hook, antes de retornos condicionais, seguindo suas regras.
- Usar chaves estáveis de identidade em listas. Não usar índice quando os itens podem mudar de ordem, nem gerar UUID durante cada render.
- Extrair componente quando existir responsabilidade, interação ou reutilização clara; evitar fragmentar cada elemento HTML em uma abstração.
- Custom hooks encapsulam comportamento compartilhado; funções puras não precisam virar hooks.

Fundamentos: [pureza de componentes e hooks](https://react.dev/reference/rules/components-and-hooks-must-be-pure) e [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks). O padrão `use` do React tem regras próprias; não generalizar suas exceções para `useEffect` ou `useState`.

Uma tabela financeira pode receber dados e callbacks; um hook da funcionalidade pode coordenar consulta e atualização. Não exigir um par “container/presentation” para todo componente se isso apenas duplicar arquivos.

## 3. Estado: proprietário e ciclo de vida

| Tipo de informação | Lugar recomendado |
| --- | --- |
| Campo em edição, expansão, diálogo | Estado local do componente ou formulário. |
| Página, filtro e ordenação compartilháveis | URL, quando houver roteamento e contrato de navegação. |
| Dados persistidos no servidor | Camada de consulta; definir atualização, invalidação e identidade da consulta. |
| Sessão visível e preferências transversais | Context/provider pequeno ou solução equivalente, quando implementado. |
| Total derivado de dados já disponíveis | Cálculo durante render ou função pura; memoização somente quando necessária. |

Não manter cópias redundantes que exigem sincronização manual. Agrupar valores que mudam juntos e explicitar estados mutuamente exclusivos, conforme [Choosing the State Structure](https://react.dev/learn/choosing-the-state-structure).

Exemplo ilustrativo de estado remoto, sem biblioteca adicional:

```ts
type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string; traceId?: string };
```

Um reducer é útil quando existem várias transições relacionadas. Uma biblioteca de cache de consultas pode ajudar quando houver invalidação, deduplicação e múltiplas telas; ela não é requisito do template atual. Redux ou outro estado global precisam de justificativa concreta.

Ao adicionar cache, incluir casa/usuário e parâmetros relevantes na identidade da consulta. Limpar dados privados no logout e impedir que a troca de casa mostre resultados da anterior. Não permitir que persistência no navegador guarde dados financeiros sem decisão explícita de produto e segurança.

## 4. Effects e chamadas assíncronas

Usar Effects para sincronização com sistemas externos. Eventos como salvar, excluir e confirmar fechamento devem disparar a ação a partir do evento do usuário. Não usar Effect para calcular valor derivável de props ou para reagir a uma flag que apenas representa um clique. Referência: [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect).

Quando uma consulta depender de parâmetros, cancelar a requisição anterior com `AbortController` e/ou invalidar sua resposta para impedir que um resultado antigo sobrescreva o atual. Executar cleanup de subscriptions e timers. Declarar dependências reais, sem desativar a regra para esconder um loop. O StrictMode pode repetir o ciclo de configuração/limpeza em desenvolvimento; a solução é tornar a sincronização correta, conforme [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects).

O Oxlint atual já verifica regras de hooks, mas o projeto ainda deve avaliar/configurar a verificação de dependências de Effects suportada por sua versão. Não assumir que `npm run lint` cobre todos os problemas de estado e assincronismo.

## 5. TypeScript e contratos

**A adotar:** habilitar `strict: true` nos projetos TypeScript da aplicação e das ferramentas. RECOMENDADO avaliar também `noUncheckedIndexedAccess` e `exactOptionalPropertyTypes`, corrigindo os erros numa mudança dedicada. Essas opções não estão todas implicitamente contidas em `strict`. Referências: [strict](https://www.typescriptlang.org/tsconfig/strict.html), [noUncheckedIndexedAccess](https://www.typescriptlang.org/tsconfig/noUncheckedIndexedAccess.html) e [exactOptionalPropertyTypes](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html).

- Usar `unknown` nas fronteiras sem confiança e refinar o tipo antes de usar.
- Não tratar `as UserResponse` como validação do JSON: assertions desaparecem em runtime.
- Definir contratos de ausência, `null`, códigos de erro e enums; não esconder divergências com `any` ou `!`.
- Usar uniões discriminadas para estados/variantes; evitar objetos com dezenas de propriedades opcionais contraditórias.
- Validar respostas críticas e dados importados em runtime. Uma biblioteca de schema é opção, não dependência já escolhida.
- Se gerar tipos a partir do OpenAPI, fixar a ferramenta, revisar o diff e manter um caminho de regeneração reproduzível. Tipos gerados não substituem autorização nem validação de runtime.

## 6. Integração HTTP

Uma camada pequena de transporte DEVE centralizar base URL, headers, cancelamento e interpretação consistente de erro. Contratos de cada recurso ficam na funcionalidade. Não criar um “serviço universal” com decisões de todas as telas.

Seguir o [padrão de APIs síncronas](API-SINCRONA.md): request sem envelope, resposta conforme o recurso, paginação uniforme e `204` sem leitura de JSON. O contrato alvo de erro inclui `code`, `errors`, `traceId` e `correlationId`; tratar sua ausência enquanto a API atual não tiver sido adaptada. Capturar `X-Correlation-Id` para suporte quando disponível, conforme o [guia de observabilidade](OBSERVABILIDADE.md).

Tratar separadamente falha de rede, cancelamento, status HTTP de erro e JSON inválido. `fetch` não rejeita apenas porque recebeu `400` ou `500`; verificar o status. Não tentar ler JSON de uma resposta `204`. Mensagens visíveis devem ser úteis e sanitizadas; quando disponível, manter `traceId` para suporte. Referência: [MDN — Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch).

Convenções alinhadas ao [guia backend](BACKEND.md): `400` apresenta validações; `401` conduz ao fluxo de sessão; `403` apresenta falta de permissão; `409` pede atualização/reconciliação, sem sobrescrever; `503` oferece recuperação adequada. Identity também prevê `422` para política de senha e `423` para bloqueio temporário. O cliente deve se apoiar em códigos estáveis, não no texto de `detail`.

No container atual, `/api/v1/...` é encaminhado pelo Nginx para `/v1/...`. O Vite ainda não tem esse proxy: configurá-lo quando a integração começar. Usar mesma origem simplifica a configuração local, mas não implementa autenticação nem CSRF.

Consultas podem ter retry limitado quando seguro; gravações não devem ser repetidas automaticamente sem o contrato de idempotência do backend. Impedir cliques duplicados melhora a experiência, mas a garantia persistente é do servidor. Para o MVP, aguardar o resultado da operação síncrona; não inventar polling de job, WebSocket ou resposta `202` para processamento não existente.

## 7. Valores financeiros e datas

O backend é a autoridade do cálculo. A interface pode apresentar prévias identificadas como tal, sem substituir validações, snapshots e arredondamento oficial.

**Proposta de contrato financeiro:** receber/enviar valores decimais como texto canônico, por exemplo `"1234.56"`, e formatá-los apenas para apresentação. A adoção precisa ser alinhada com o backend antes do primeiro endpoint financeiro. Alternativa: inteiros em centavos, somente se os limites do produto garantirem `Number.isSafeInteger`; o alcance de `numeric(18,2)` não cabe integralmente nessa hipótese.

- Não usar `parseFloat` de entrada localizada nem ponto flutuante binário para fechar contas.
- Separar texto digitado, valor validado e apresentação `pt-BR`/BRL.
- `Intl.NumberFormat` formata; não define precisão de cálculo nem faz parsing de moeda.
- Para cálculo exato no cliente, escolher uma representação/biblioteca decimal compatível e testar os mesmos exemplos do contrato. Não escolher uma dependência antes de existir esse requisito.
- Não converter uma string decimal grande para `Number` apenas para formatar se isso perder precisão.
- `edit_version` e contadores `bigint` exigem representação textual ou limites seguros definidos no contrato. Não alterar silenciosamente o `totalCount` numérico já existente.
- Competência `YYYY-MM` ou data civil não deve passar por uma conversão de fuso que troque o mês. Instantes UTC podem ser exibidos no fuso definido para a experiência.

As escolhas persistentes estão no [modelo físico](../db/DB-Modelo-Fisico.md); o formato financeiro da API permanece uma decisão a formalizar.

## 8. Formulários e acessibilidade

Usar HTML semântico: `button` para ação, `a` para navegação, labels associados a campos e cabeçalhos corretos em tabelas. Mensagens de erro devem identificar campo e correção, ligadas por `aria-describedby` quando necessário. Preservar valores válidos após falha e levar o foco ao resumo/primeiro erro conforme a interação. Referência: [W3C — Forms Tutorial](https://www.w3.org/WAI/tutorials/forms/).

**Meta do projeto para novas telas:** WCAG 2.2 nível AA, validada por testes e revisão manual; não é certificação do template atual. Garantir teclado, foco visível, contraste, ampliação, nomes acessíveis e informação que não dependa só de cor. Diálogos devem administrar foco e permitir retorno ao acionador. Usar notificações de status acessíveis sem repetir anúncios a cada render. Referência: [WCAG 2.2 Quick Reference](https://www.w3.org/WAI/WCAG22/quickref/).

Toda tela remota DEVE tratar carregamento, vazio, erro e sucesso; operações de escrita também precisam de estado pendente. Testar resolução pequena, conteúdo longo e navegação por teclado. Bibliotecas de UI não garantem acessibilidade automaticamente.

## 9. Segurança, sessão e configuração

O HLD/FDD prevê Cognito acessado pelo backend e cookies `HttpOnly`, `Secure` e `SameSite`. Tokens de sessão não devem ficar em `localStorage`, `sessionStorage`, URLs ou logs. No futuro fluxo, JavaScript lê o estado permitido da sessão por endpoint; não precisa ler o token. Referência: [OWASP — Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).

CSRF deve ser resolvido junto do backend para requisições autenticadas por cookie. SameSite é uma camada de defesa, não uma solução universal. Definir token/verificação de origem e comportamento de credenciais conforme os domínios usados. Referência: [OWASP — CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).

Não renderizar HTML não confiável com `dangerouslySetInnerHTML`; se houver requisito real de conteúdo rico, definir sanitização e política de origens. Ocultar botões é conveniência de interface, não autorização. Não registrar senha, token ou payload financeiro completo em console/telemetria.

`VITE_*` é configuração pública embutida no bundle. Nunca incluir senhas, segredo Cognito ou chaves privadas. Variáveis do container Nginx não modificam automaticamente um bundle já compilado. Referência: [Vite — Env Variables and Modes](https://vite.dev/guide/env-and-mode).

## 10. Desempenho e dependências

Medir antes de otimizar. Dividir bundle por rota/funcionalidade pesada quando houver benefício; avaliar virtualização apenas para listas que realmente precisem. Paginação deve começar no contrato da API, não baixando todos os dados para cortar no navegador.

`useMemo`, `useCallback` e `memo` não são regras para todo componente. Usá-los quando identidade de referência ou custo medido justificar. O React Compiler não está habilitado no template; não presumir sua otimização. Evitar cadeias de Effects que causam renders adicionais por estado derivado.

Uma dependência nova deve ter propósito, compatibilidade com React/TS/Vite usados, manutenção, licença, custo de bundle e alternativa avaliados. Manter `package-lock.json`; usar `npm ci` em instalação reproduzível. Atualizações importantes devem passar por build, lint e testes da funcionalidade afetada.

## 11. Testes e revisão

**Ferramentas propostas, ainda não instaladas:** Vitest para funções/componentes, React Testing Library para interação e Playwright para poucos fluxos E2E críticos. Validar compatibilidade antes de fixar versões. Referências: [Vitest](https://vitest.dev/guide/), [Testing Library — Guiding Principles](https://testing-library.com/docs/guiding-principles/) e [Playwright — Best Practices](https://playwright.dev/docs/best-practices).

Testar comportamento observável, consultando por papel/nome acessível; evitar depender de estado interno, classes CSS ou snapshots extensos. Cobrir parsing financeiro, validação, estados da tela, erro/retry, troca de casa, cancelamento e conflito. Testes E2E usam dados isolados e esperas por condições, sem sleeps fixos nem dependência da ordem dos testes.

Comandos existentes, a partir de `expenses-ui/`:

```bash
npm ci
npm run lint
npm run build
```

Não existe `npm test` configurado atualmente. Adicionar scripts e instruções junto da adoção do runner; não descrever comandos futuros como checks disponíveis.

- [ ] Componentes coesos, renderização pura e estado com proprietário claro.
- [ ] Tipos e validação das fronteiras tratados sem assertions que ocultem erro.
- [ ] Respostas antigas/canceladas não sobrescrevem o estado atual.
- [ ] Carregamento, vazio, erro, pendência e sucesso cobertos.
- [ ] Dinheiro, datas e conflitos seguem o contrato da API.
- [ ] Teclado, foco, labels e mensagens verificados.
- [ ] Nenhum segredo no bundle; cache e sessão isolam usuários/casas.
- [ ] Lint, build e testes disponíveis passaram; ferramentas pendentes identificadas.
