# Git Flow - Padrão de Branches e Processo

## Branches Principais (Protegidas)
- **main**: Sempre estável, representa o código em produção.
- **hlg**: Branch de homologação (QA para testes e homologação).
- **dev**: Branch de desenvolvimento integrado (sandbox de desenvolvimento).

## Tipos de Branches e Nomenclatura
- **feature/**: Branch de uma história. Sempre criada a partir de `main`.
	- Padrão: `feature/<id-historia>`
	- Exemplo: `feature/123`
- **task/**: Branch de uma subtarefa da história. Sempre criada a partir da respectiva `feature`.
	- Padrão: `task/<id-subtask>`
	- Exemplo: `task/456`
- **merge/**: Branch intermediária para resolver conflitos complexos.
	- Padrão: `merge/<branch-origem>-into-<branch-destino>`
	- Exemplo: `merge/feature-123-into-dev`
- **fix/**: Branch para correções emergenciais em produção, criada a partir de `main`.
	- Padrão: `fix/<id-tarefa>`
	- Exemplo: `fix/789`

## Fluxo de Trabalho

1. **Início de uma História**
	 - Criar uma branch `feature/` a partir da `main`.
	 - Exemplo: `feature/123`

2. **Desenvolvimento de Subtarefas**
	 - Para cada subtarefa, criar uma `task/` a partir da `feature/`.
	 - Cada `task/` deve abrir PR/MR de volta para a `feature/`.
	 - Isso permite code review granular e mantém a feature como referência da história.
	 - Exemplo:
		 - `feature/123`
		 - `task/456`
		 - `task/457`

3. **Integração em Dev**
	 - Quando a feature estiver pronta para integração:
	 - Criar PR/MR da `feature/*` → `dev`.
	 - Aqui ocorre integração com outras features e testes internos.

4. **Homologação**
	 - Após validação em `dev`, promover a mesma feature branch para `hlg`.
	 - PR/MR: `feature/*` → `hlg`.
	 - Sempre que houver um bug em homologação, efetuar a correção na feature branch, nunca diretamente em `hlg`.
	 - Nunca fazer merge de `dev` direto em `hlg`.

5. **Produção**
	 - Com a homologação aprovada, promover a mesma feature branch para `main`.
	 - PR/MR: `feature/*` → `main`.
	 - Nunca fazer merge de `hlg` direto em `main`.

6. **Hotfix em Produção**
	 - Caso um problema seja encontrado em produção:
		 - Criar branch `fix/` a partir de `main`.
		 - Após correção, abrir PR/MR para:
			 - `main` (corrigir produção imediatamente)
			 - `hlg` (manter homolog atualizado)
			 - `dev` (manter integração coerente)
