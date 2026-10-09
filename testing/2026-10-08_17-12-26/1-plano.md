# E5 — Interface de autenticação

## Identificação
Abertura: 2026-10-08 17:12:26 America/Sao_Paulo. Responsável: desenvolvimento/Codex.
Escopo: mudanças locais da [FDD de autenticação](../../fdd/FDD-Criacao-Usuario-Autenticacao/FDD-Criacao-Usuario-Autenticacao.md), etapa E5. [Execução](2-execucao.md).

## Objetivo e limites
Comprovar cadastro, login, consulta/renovação e saída pela interface, usando a API implementada. Sem Household, recuperação de senha ou alterações de schema. Identidade local simulada não homologa Cognito nem a meta p95 real.

## Inventário e matriz
Rotas visuais: /login, /cadastro, / (identidade autenticada). Endpoints: POST users, GET csrf, POST sessions, GET session/current, DELETE sessions/current sob /api/v1/identity.

| ID | Preparação/ação e camada | Esperado |
| --- | --- | --- |
| UNIT-UI | Dependências substituídas; campos inválidos, senha, envio duplo, armazenamento, erro, retry de cadastro, logout indisponível | Foco correto; sem chamada inválida/duplicada; apenas e-mail opt-in; mesma chave na repetição; sem falsa confirmação de saída |
| UI-01 | Navegador novo; abrir login desktop/mobile | Formulário acessível, sem overflow; capturas para revisão |
| UI-02 | E-mail sintético novo; cadastrar pela tela, entrar, recarregar, sair | Cadastro sem login automático; sessão restaurada; identidade removida após saída; senha/tokens ausentes do storage |
| UI-03 | Conta sintética; dez senhas inválidas; outra conta válida | Avisos 3/2/1; bloqueio por e-mail; outra conta funciona |
| UI-04 | Conta sintética, marcar lembrar e-mail; logout/recarga/desmarcar | Só e-mail persistido e removido ao desmarcar |
| UI-05 | Duas abas autenticadas, sair em uma | Outra aba perde identidade e sessão copiada é revogada |
| UI-06 | Usuário duplicado no cadastro, formulário vazio por teclado | Mensagem de duplicidade; foco e validação antes de API |
| API/BROWSER | Reexecutar suíte existente com frontend novo | Nove cenários HTTP e dois de segurança de cookies entre sites preservados |

## Ambiente e dados
Node 24, React 19, Chromium do Playwright, duas APIs .NET 10, PostgreSQL 17 descartável e Liquibase. Certificado HTTPS exclusivo do laboratório; provedor sintético, sem AWS. Serviços de desenvolvimento permanecem parados; banco pessoal preservado.

## Execução e critérios
Build, lint, Vitest, tipagem do projeto de testes e `scripts/lab.py all`. Todos os casos do ciclo devem passar. Capturas desktop/mobile revisadas. Não executar carga ou scan completo por mudança somente da interface; resultados anteriores e pendências de performance/segurança continuam válidos, sem novo aceite. Firefox/WebKit ficam pendentes de campanha específica.

## Evidências e limpeza
Relatórios sanitizados em expenses-tests/reports/<run-id>. Limpeza automática em finally: processos, banco, container e chaves privadas do laboratório. Confirmar relatório e ausência do container.

## Riscos e revisões
Compose HTTP atual não suporta cookies Secure para autenticação; uso real exige HTTPS e configuração privada do provedor. Sem declarar FDD concluída ou apta para produção. Plano aberto durante implementação, antes da campanha; primeira compilação identificou sintaxe TypeScript incompatível, corrigida sem alterar contrato.
