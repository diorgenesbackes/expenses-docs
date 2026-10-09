# 2 — Execução: testes locais

Registro documental: **2026-10-08 12:02:50 — America/Sao_Paulo**. Histórico migrado, preservando datas/IDs e resultados originais; esta data não indica nova execução.

[Padrão e Definition of Done](../README.md) · [Plano](1-plano.md) · [Execução](2-execucao.md) · [Histórico da FDD](../../fdd/FDD-Criacao-Usuario-Autenticacao/3-testes.md)

Data: 2026-10-08. **Base de testes implementada e validada localmente; não constitui aceite de produção.** Nenhum recurso AWS foi criado e os três testes Cognito não foram executados nesta entrega.

## Resultado funcional

| Grupo | Resultado desta execução | Onde fica |
| --- | ---: | --- |
| Backend unitário | 46 aprovados, zero ignorados | `expenses-service/Expenses.UnitTests` |
| Frontend unitário | 2 aprovados | `expenses-ui/src/App.test.tsx` |
| HTTP em processo + PostgreSQL real | 49 aprovados, zero ignorados | `expenses-tests/integration` |
| API por HTTPS/Kestrel | 9 aprovados | `expenses-tests/api` |
| Chromium real | 3 aprovados | `expenses-tests/e2e` |
| Liquibase | Upgrade, reaplicação, integridade, rollback/reaplicação parcial e total aprovados | `expenses-tests/integration/schema` |
| Cognito real | 3 preservados/compilados, não executados | `expenses-tests/external` |

São **109 casos automatizados locais aprovados**, além do ciclo de validação do schema. Os 98 originais continuam identificados individualmente por classe no manifesto: 46 unitários, 49 integrações, três externos. Não chamar a reorganização de “98 unitários” nem anunciar os três externos como aprovados nesta execução.

Evidência consolidada: [resultado](../../../expenses-tests/reports/checks-cf5781f7ad1c/summary-corrected.json), [49 integrações](../../../expenses-tests/reports/112c252b9f0d/summary.json), [API/navegador](../../../expenses-tests/reports/5c99d11b19ac/summary.json). O agregado original foi preservado; uma associação de diretório concorrente foi corrigida a partir dos logs de cada runner, sem alterar resultados de testes. O orquestrador agora associa somente relatórios anunciados pelo próprio subprocesso.

Artefatos em `reports/` são locais, privados e ignorados no Git; os links deixam de funcionar se forem removidos. Este documento registra os resultados essenciais para revisão/versionamento.

## O que o laboratório comprova

- O mesmo `Program` da API roda em Kestrel real, com duas instâncias em processos separados, PostgreSQL real e Liquibase.
- Cadastro válido, replay, duplicidade e concorrência têm resultados verificáveis pela rede.
- Login normaliza o e-mail; dez falhas bloqueiam entre instâncias, sem bloquear outra conta pelo mesmo IP.
- Cookies HttpOnly/Secure/Strict, CSRF, origem e limitação de corpo funcionam no host publicado.
- Consulta em outra instância, renovação e revogação imediata rejeitam cookies copiados após logout.
- JWT adulterado é rejeitado. O provedor local emite RS256; a validação criptográfica real continua ativa.
- Indisponibilidade do provedor não consome tentativas de senha. Banco pausado falha fechado e recupera quando retorna.
- Chromium abre a aplicação, interage com o contador, não lê o cookie HttpOnly e não envia a sessão Strict numa navegação de iframe entre sites.
- Recursos descartáveis foram removidos após sucesso e após falhas; dados de desenvolvimento não foram utilizados.

O simulador é exclusivo de `expenses-tests`, sem opção de fallback na API de produção. Banco, chaves e estado são compartilhados apenas entre as instâncias daquele laboratório. A aplicação de produção não recebeu substituições de autenticação nem alterações de schema nesta entrega.

## Baseline de desempenho

Três execuções finais em Node 24.15.0, build Release, .NET SDK 10.0.401, PostgreSQL 17.11 e k6 2.3.0. Cada execução criou 1.000 perfis sintéticos fora da janela medida e coletou 1.000 observações de cada operação. Identidade simulada, duas instâncias locais; gerador e banco com 1 CPU/512 MB cada. API/proxy são processos do host, sem limite rígido de recursos. Máquina: macOS 26.6.2 arm64. Aquecimento separado da janela medida.

| Operação | p95 — execução 1 | p95 — execução 2 | p95 — execução 3 |
| --- | ---: | ---: | ---: |
| Cadastro | 7 ms | 8 ms | 8 ms |
| Login | 7 ms | 7,05 ms | 7 ms |
| Consulta | 2 ms | 2 ms | 2 ms |
| Renovação | 4 ms | 4 ms | 4 ms |
| Logout | 3 ms | 3 ms | 3 ms |

Execuções: [53bb8ee9a468](../../../expenses-tests/reports/53bb8ee9a468/summary.json), [2be8d499efb5](../../../expenses-tests/reports/2be8d499efb5/summary.json), [af6ddc1da2c0](../../../expenses-tests/reports/af6ddc1da2c0/summary.json). Cada pasta contém `k6.json` com p50/p95/p99, quantidades e critérios, além de recursos em `resources.jsonl`. A terceira também identifica os fontes por hash/estado Git; o runner mantém essa identificação nas próximas execuções.

Os critérios locais de p95 < 150 ms, amostras mínimas e respostas esperadas passaram. Os tempos completos são medidos no cliente com resolução de milissegundos. **Não substituem a evidência Cognito anterior**, cujo cadastro ficou em aproximadamente 1.203 ms no p95. Não comprovam latência de produção, capacidade para 1.000 simultâneos ou o comportamento de inicialização fria por operação.

Perfis implementados e verificados pelo `k6 inspect`: sanidade de 2 min, baseline, consultas em 1→5→10 req/s, jornadas de 10→25→50 usuários, pico de 100 e duração de uma hora. Nesta entrega foi executada a baseline; os demais perfis ainda não têm aprovação empírica. As taxas/concorrências são hipóteses exploratórias. Ainda faltam ensaio de estresse com parada por saturação, separação automática de refresh nas séries longas e observabilidade interna de pool SQL/tracing.

## Segurança e triagem

Auditorias npm/NuGet passaram sem vulnerabilidades conhecidas no momento da execução. Gitleaks 8.30.1 passou após confirmar um falso positivo: o UUID de exemplo de `Idempotency-Key` em `API-SINCRONA.md`. A exceção exige simultaneamente aquele caminho e aquele valor; não exclui a documentação inteira. [Relatório](../../../expenses-tests/reports/security-d3c7bb28ec27/summary.json).

ZAP 2.17.0 passivo e ativo foram executados contra o laboratório. O scan ativo registrou resposta **200 na rota protegida**, além dos checks autenticados antes/depois. Nenhum achado médio, alto ou crítico foi reportado nesse escopo. Permanecem quatro tipos de alerta baixo; o runner retorna falha enquanto a triagem não estiver resolvida, sem suprimir avisos. [Relatório ativo redigido](../../../expenses-tests/reports/8aad64b9470b/zap.json).

| Alerta | Triagem | Próximo tratamento |
| --- | --- | --- |
| CORP ausente — 90004 | Ausência confirmada em health/probe; hardening de resposta pendente. | Definir política compatível com origens legítimas e implementá-la na composição de entrega. |
| HSTS ausente — 10035 | Ausência confirmada no laboratório localhost; não valida a política de transporte da publicação. | Configurar/verificar HSTS no domínio HTTPS de destino, considerando o efeito sobre outros serviços do mesmo host. |
| `X-Content-Type-Options` ausente — 10021 | Ausência confirmada em health/probe. | Aplicar `nosniff` na resposta apropriada e verificar o proxy final. |
| Content-Type inesperado — 100001 | O scanner de API também consultou a raiz `/`, que serve legitimamente o HTML do frontend. | Separar contexto/API e contexto/frontend no scanner; não alterar o HTML para simular uma API nem ignorar globalmente Content-Type. |

Os primeiros três itens continuam abertos; o quarto tem explicação de escopo, mas não foi ocultado automaticamente. O teste de probe existe somente no host de testes e não representa uma API de negócio pública.

A varredura ativa encontrou um defeito real do **proxy exclusivo do laboratório**: uma tentativa de servir um diretório podia escrever headers duas vezes e encerrar o processo. A leitura passou a ocorrer antes dos headers, entradas inválidas passaram a retornar erro controlado e API-09 cobre a recuperação. O scan foi repetido; a execução anterior com falha permanece registrada. Também foram preservadas falhas iniciais de confiança TLS/rota Docker e a primeira tentativa inadequada de observar CORS no teste SameSite. Não foram tratadas como resultados aprovados.

Os checks de segurança atuais cobrem parte dos temas de autenticação, sessão, controle de acesso, validação e configuração do ASVS. **Não há ainda mapeamento completo por requisito ASVS nível 2**, revisão independente ou aceite de política de senha/MFA/e-mail não verificado. Login, refresh e logout usam testes dedicados; o scan não os exercita dinamicamente para evitar revogar a própria sessão. Ampliar esse contexto exige controle de sessão do scanner.

## Decisões e pendências de implantação

| Etapa do plano | Situação |
| --- | --- |
| T0/T1 — inventário e separação | Concluída e validada; comandos antigos encaminham para o projeto novo. |
| T2 — laboratório | Funcional: HTTPS, duas instâncias, identidade local, banco isolado e limpeza. API/proxy ainda são processos locais; Compose completo e Nginx de destino permanecem pendentes. |
| T3 — API pela rede | Nove cenários prioritários aprovados; ampliar matriz de restart/rotação de chaves, corpo chunked pela rede e demais limites conforme evolução. |
| T4 — frontend | Runner unitário e navegador configurados; autenticação visual aguarda as telas E5. Apenas Chromium foi executado. |
| T5 — performance | Baseline tripla aprovada localmente; perfis longos preparados, ainda não executados. |
| T6 — segurança | Auditorias e scans executados; hardening, contexto ampliado, ASVS detalhado e revisão de imagens ainda pendentes. |
| T7 — nuvem | Adiada conforme decisão do usuário; três testes Cognito preservados, opt-in. |

Segurança/performance de produção não estão homologadas. Próxima execução de carga deve usar uma máquina disponível sem outras suítes, começar por sanidade e carga gradual e só então avançar para pico/duração. Uma hora de teste não deve ser substituída por um ensaio curto rotulado como soak.

## Operação

Ver o [README do novo projeto](../../../expenses-tests/README.md) para pré-requisitos, comandos, versões, limites e tratamento de relatórios. O comando `python3 scripts/check.py` executa o ciclo local consolidado sem carga e sem ZAP; `--zap` acrescenta a varredura passiva e torna os alertas visíveis no resultado agregado. Cognito nunca é acionado por esses comandos.
