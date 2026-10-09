# 3 — Testes: criação de usuário e autenticação

Este documento relaciona a FDD aos seus ciclos e às evidências históricas. Os planos/resultados detalhados ficam em `testing/`, sem cópias concorrentes. Organização documental: 2026-10-08; nenhum teste novo foi executado nesta reorganização.

[Especificação](FDD-Criacao-Usuario-Autenticacao.md) · [Desenvolvimento](2-desenvolvimento.md) · [Padrão e Definition of Done](../../testing/README.md)

## Ciclos documentados

| Registro | Plano | Execução | Escopo e situação |
| --- | --- | --- | --- |
| 2026-10-08_12-02-50 | [1-plano](../../testing/2026-10-08_12-02-50/1-plano.md) | [2-execucao](../../testing/2026-10-08_12-02-50/2-execucao.md) | Base local: unidades, integrações, API pela rede, navegador, baseline e segurança. Entrega parcial em relação à FDD; UI de autenticação e homologação ainda pendentes. |
| 2026-10-08_17-12-26 | [1-plano](../../testing/2026-10-08_17-12-26/1-plano.md) | [2-execucao](../../testing/2026-10-08_17-12-26/2-execucao.md) | E5: 11 unitários e 17 testes HTTP/navegador aprovados; interface validada localmente com identidade simulada. |

A pasta datada acima identifica a organização do histórico existente; as datas e os IDs dos testes originais permanecem no documento de execução. Novos ciclos devem receber novas pastas conforme o padrão.

## Evidências anteriores com Cognito real

| Evidência | O que registra | Limite |
| --- | --- | --- |
| [Latência do cadastro](3-testes-latencia-cadastro.json) | Vinte amostras em 2026-10-07; p95 aproximadamente 1.203 ms. | Acima da meta de 150 ms; medição exploratória, não homologação. |
| [Latência das sessões](3-testes-latencia-sessoes.json) | Uma amostra por operação em 2026-10-07, incluindo login, refresh e logout. | Não permite afirmar p95. |

Os registros funcionais originais de E2 e E3/E4 foram preservados em [2-desenvolvimento](2-desenvolvimento.md). Os JSONs foram apenas movidos/renomeados, sem alterar as medições. A baseline com identidade simulada não substitui os resultados Cognito.

## Situação do Definition of Done

**A FDD não está concluída.** O backend e a base de testes têm entregas validadas, e E5 agora possui telas e jornadas locais aprovadas. Permanecem pendentes homologação no ambiente de destino, critérios de desempenho com dependências/ambiente de destino e os itens de segurança/carga descritos na execução. Não considerar a reorganização da documentação uma aprovação de requisitos ou de publicação.

Ao concluir um novo ciclo, acrescentar sua referência à tabela e revisar essa situação a partir das evidências, preservando o histórico.
