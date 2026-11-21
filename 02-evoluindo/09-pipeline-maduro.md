# 09 — Evoluindo: Pipeline maduro

**Onde estamos:** pipelines completos em ideia (teste, build, deploy, segurança, eval de prompts), mas a maioria dos passos não pode reprovar, os passos de deploy estão duplicados e tudo usa `latest`.

---

## 🥉 Nível 1 — Deixar os gates reprovarem

1. **Remova os `|| true`** de lint e testes. Se quebrar, o deploy não acontece.
2. **Trivy com código de saída:** `trivy fs --exit-code 1 --severity CRITICAL .` — começa só com CRITICAL para não travar tudo de uma vez.
3. **Smoke test que falha de verdade:** `curl -sf ... || exit 1`.
4. **Espera do rollout sem `|| true`** — se não subir, o pipeline deve ficar vermelho.
5. Se algo ainda não pode bloquear, deixe isso **explícito** (`continue-on-error: true` com um comentário explicando o porquê e até quando).

> Um passo que nunca falha é só um passo que gasta tempo.

---

## 🥈 Nível 2 — Menos repetição, mais clareza

- **Um único fluxo de deploy**, chamado pelo push e pelo botão manual. Opções: um *reusable workflow* (`workflow_call`) ou uma *composite action*.
- **Tag de imagem = commit** (ver [02](02-imagens-e-versoes.md)), passada entre jobs por `outputs`.
- **Destroy honesto:** `kubectl delete namespace <ns>` (apaga tudo, inclusive PVCs) — com confirmação.
- **Concorrência:** evite dois deploys simultâneos:
  ```yaml
  concurrency:
    group: deploy-${{ github.ref }}
    cancel-in-progress: false
  ```
- **Permissões mínimas** no token do workflow (`permissions: contents: read`).
- **Cache** de dependências (`actions/setup-python` com `cache: pip`).

---

## 🥇 Nível 3 — Entrega segura

- **Ambientes do GitHub** (`environment: production`) com aprovação manual e segredos separados.
- **Deploy via GitOps** (ver [04](04-gitops.md)): o pipeline só faz build, scan e atualiza a tag.
- **Rollout progressivo** (canary/blue-green) com Argo Rollouts ou Flagger, voltando sozinho se as métricas pioram.
- **Branch protection** exigindo os checks verdes (incluindo o eval de prompts e o scan de segurança) para fazer merge.
- **Actions fixadas por hash** de commit, atualizadas pelo Dependabot.
- **Runner efêmero** (um container novo por job) em vez de uma máquina reaproveitada.

---

## Como validar

Abra um PR que **quebra um teste de propósito**. O resultado esperado:
- PR com check vermelho;
- merge bloqueado;
- nada deployado.

Se algum desses não acontecer, ainda há um gate que não reprova.
