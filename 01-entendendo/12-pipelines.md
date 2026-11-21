# 12 — Pipelines (GitHub Actions)

## O que é?

Um **pipeline** é uma sequência automática de passos disparada por um evento (push, pull request, botão manual). No GitHub, isso se chama **GitHub Actions** e fica em `.github/workflows/`.

- **CI (Integração Contínua)**: a cada mudança, testar e validar.
- **CD (Entrega/Deploy Contínuo)**: depois de validado, colocar no ar.

> **Analogia:** uma linha de montagem com inspetores. Cada estação (job) faz uma coisa; se um inspetor reprova, a peça não segue.

## Vocabulário

| Termo | Significado |
|---|---|
| `on:` | Gatilho (push, pull_request, workflow_dispatch = botão manual) |
| `jobs:` | Etapas grandes; rodam em paralelo, a menos que tenham `needs:` |
| `needs:` | "Só comece depois daquele job" |
| `steps:` | Comandos dentro de um job, em sequência |
| `runs-on:` | Em que máquina roda |
| `uses:` | Reaproveita uma ação pronta (ex.: `actions/checkout`) |
| `secrets.X` | Valores sensíveis guardados no GitHub, não no código |
| `if:` | Condição para rodar |

## Conceito: runner *self-hosted*

Todos os workflows do app-api rodam em `self-hosted`: **uma máquina sua** (provavelmente o mesmo computador do cluster k3d), não um servidor do GitHub. Vantagem: o pipeline enxerga o cluster local e o modelo de IA local. Risco: qualquer código de um PR roda **na sua máquina** — em repositórios públicos isso é perigoso.

## Os 4 workflows, por conceito

### 1. Deploy (push na main + botão manual)
Cadeia **teste → build → deploy**:
1. Instala dependências, roda lint e testes.
2. Constrói a imagem com três tags (latest, hash do commit, número do build).
3. Importa a imagem no k3d (não há *registry*; a imagem é copiada direto para o cluster).
4. `kubectl apply` na pasta inteira, espera o rollout e faz um **smoke test** (chamada rápida para ver se responde).

O botão manual oferece **deploy** ou **destroy** (apagar o ambiente).

### 2. Segurança (em pull requests)
- **SonarQube**: análise estática do código (bugs, *code smells*, vulnerabilidades).
- **Trivy**: procura vulnerabilidades em dependências (`fs`) e más configurações nos YAMLs do Kubernetes (`config`).
- **Agente de segurança com IA**: lê os resultados dos dois e comenta no PR.

### 3. Eval gate (PR que mexe em prompts)
Roda avaliações de prompts com Promptfoo contra um LLM local e **bloqueia o merge** se a taxa de acerto ficar abaixo de 85%. É o conceito de **quality gate** aplicado a IA.

### 4. Validação de OTel (em PRs)
Lint + testes específicos da instrumentação de telemetria. **Este é o único workflow cujos passos realmente falham quando algo dá errado.**

## O conceito mais importante: um gate precisa poder reprovar

Repare quantos passos terminam com `|| true` ou "se falhar, só avise":

- lint, testes, quality gate do Sonar, Trivy, espera do rollout do banco, smoke test...

`|| true` significa "**independentemente do resultado, considere sucesso**". Um inspetor que nunca reprova não é inspetor. Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## Outros pontos de atenção

- Os passos de deploy estão **copiados** entre o job automático e o manual. Mudar um e esquecer o outro é questão de tempo.
- O "destroy" usa `kubectl delete all`, que **não** apaga ConfigMap, Secret, PVC, Ingress, NetworkPolicy etc. — "all" no kubectl não é tudo.
- Ações referenciadas por tag (`@v4`) em vez de hash de commit.

Próximo: [13 — Observabilidade](13-observabilidade.md)
