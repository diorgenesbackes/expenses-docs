# Execução E5 — Interface de autenticação

## Identificação
[Plano](1-plano.md) · [FDD](../../fdd/FDD-Criacao-Usuario-Autenticacao/FDD-Criacao-Usuario-Autenticacao.md).
Responsável: desenvolvimento/Codex. Abertura: 2026-10-08 17:12:26 America/Sao_Paulo, status inicial **não executado**. Campanha final de navegador: 17:17:53–17:18:13, mesmo fuso. Status final: **aprovado no escopo local deste ciclo**, com pendências de homologação da FDD.

## Ambiente real
MacOS arm64; Node 24.15.0, .NET SDK 10.0.401, Playwright 1.64.0/Chromium, Vite 8.3.2, Vitest 5.0.3. Duas APIs Kestrel, proxy HTTPS exclusivo do laboratório, identidade simulada compartilhada e PostgreSQL 17.11 com 1 CPU/512 MB. Migrations aplicadas pelo Liquibase antes das requisições. Nenhuma AWS/conta real utilizada.

Fontes locais não commitados sobre backend `39c5f4e6cbae2fce980172c78443d9084bef1f77` e frontend `fa83f32437f276b706647e7841cecc939828342c`; expenses-tests ainda sem Git. Hashes da campanha em summary.json: UI `cbe58676279c6aee453b6b2d05986893981c6b21e3debf64e08bba347cbf1f76`, backend `b5736d7c89432006125aa94019e3bacf2fa60424395f4e6bb5b92beeae33990e`. Atualizações documentais posteriores não alteram a implementação ensaiada.

## Tentativas e resultados

1. Primeira compilação falhou: parameter properties incompatíveis com `erasableSyntaxOnly`; corrigidas para propriedades explícitas. Um comando inicial Node não chegou a executar por DNS restrito; repetido com acesso apropriado. Sem alteração de critério.
2. Lint inicial: avisos para expressão de caracteres de controle e bootstrap da sessão via effect. Validação alinhada com Unicode/controles do contrato e comentário restrito para sincronização necessária da sessão. Versão final sem avisos.
3. Primeiro laboratório `d6163a64be8a`: **12 passaram, 5 falharam, 0 ignorados**. UI-02 a UI-06 não encontraram os formulários: proxy do laboratório não servia diretamente `/login` e `/cadastro` (404). Corrigido fallback restrito às rotas conhecidas; preservado API-09 para caminhos/arquivos inválidos. Container e processos removidos apesar da falha.
4. Campanha final `52126ab726fa`: **17 passaram, 0 falharam, 0 ignorados**, sem retries, em 19,5 segundos de Playwright. Ambiente inteiro em 22,19 segundos após preparação do banco. Duração da suíte não equivale a p95 de operações.
5. `npm run lint`, `npm test`, `npm run build` via Node 24: aprovados; **11 unitários em 2 arquivos**. `tsc --noEmit` em expenses-tests: aprovado. Build do host .NET: zero erros/avisos. Não foi necessário repetir as 49 integrações em processo, carga prolongada ou scanner completo, pois backend/schema/dependências de runtime não mudaram.

Comando da campanha: `npm exec --yes --package=node@24.15.0 -- python3 scripts/lab.py all` em expenses-tests. Cada tentativa construiu o frontend e host, criou banco isolado, aplicou Liquibase e encerrou o laboratório.

| Cenário do plano | Resultado real |
| --- | --- |
| UNIT-UI | 11 aprovados: validação/foco, envio duplo, remoção de e-mail, mesma chave em repetição incerta, avisos de tentativas/senha apagada, logout sem confirmação falsa, política de senha, CSRF, resposta inválida, escrita sem retry, 204 e Retry-After |
| UI-01 | Login em 1280×720 e 375×812, foco no e-mail inválido, sem overflow horizontal; capturas revisadas visualmente |
| UI-02 | Cadastro visual → login → recarga → remoção do cookie de acesso/renovação por refresh → logout → recarga anônima; localStorage/sessionStorage sem credenciais |
| UI-03 | Dez tentativas visuais: avisos 3/2/1, bloqueio e entrada de outra conta sem bloqueio global |
| UI-04 | Opt-in persiste apenas e-mail; logout/recarga preservam preferência; desmarcar remove imediatamente |
| UI-05 | Saída em uma aba remove a identidade da outra; revogação do ticket copiado entre instâncias também coberta por API-02 |
| UI-06 | Cadastro duplicado gera mensagem e foco no resumo, sem autenticar |
| API/BROWSER | Nove cenários HTTP e dois de cookies/CSRF/SameSite aprovados, incluindo banco/provedor indisponíveis e recuperação |

## Evidências

- [Resumo final e hashes](../../../expenses-tests/reports/52126ab726fa/summary.json)
- [Relatório Playwright final](../../../expenses-tests/reports/52126ab726fa/playwright.json)
- [Desktop](../../../expenses-tests/reports/52126ab726fa/login-desktop.png) e [mobile](../../../expenses-tests/reports/52126ab726fa/login-mobile.png)
- [Primeira tentativa](../../../expenses-tests/reports/d6163a64be8a/playwright.json)

Artefatos locais ignorados pelo Git podem expirar. Este documento preserva os resultados essenciais. Capturas contêm apenas a tela sem dados de conta. Dados dos testes são sintéticos; sem snapshots persistentes de cookies/credenciais.

## Achados e limitações
Fallback do laboratório corrigido e retestado; Nginx já possuía fallback SPA. Nenhuma mudança de esquema ou enfraquecimento de TLS/CSRF. Senha/tokens ausentes de armazenamento web.

A revisão local não certifica WCAG integral: teclado/foco, labels, responsividade e aparência foram verificados; leitores de tela e demais navegadores continuam pendentes. Firefox/WebKit não executados. Meta p95 <150 ms com Cognito, cargas longas, achados anteriores do scanner e infraestrutura de destino continuam pendentes da FDD. O Compose de desenvolvimento atual permanece HTTP e precisa de HTTPS/configuração privada do provedor para autenticação real. Falhas operacionais da interface têm cobertura unitária; a campanha não injeta todos os erros de rede possíveis em navegador.

## Limpeza
As duas execuções removeram seus containers exclusivos (`expenses-tests-d6163a64be8a` e `expenses-tests-52126ab726fa`), banco, diretório privado de chaves e processos. `processesStopped: true` em ambos os resumos. Banco de desenvolvimento preservado. Frontend/backend de desenvolvimento continuam parados a pedido do usuário.

## Conclusão e DoD
E5 entregue e validado localmente: seis jornadas visuais, dois controles de navegador e nove cenários HTTP passaram, além de 11 unitários. Critérios deste ciclo atendidos. **FDD ainda não concluída/não homologada para publicação:** dependências e performance reais, ambiente HTTPS habitual, navegadores adicionais e pendências de carga/segurança exigem ciclos próprios. As evidências anteriores foram preservadas e não foram reclassificadas como novas aprovações.
