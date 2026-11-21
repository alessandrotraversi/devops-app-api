# Entendendo o DevOps do `app-api`

Este repositório é um **material de estudo**. Ele explica, peça por peça, as configurações de DevOps do repositório [`app-api`](https://github.com/robinho1995/app-api) — Kubernetes, Docker, pipelines e observabilidade.

Essas configurações foram geradas com ajuda de IA. Isso é ótimo para andar rápido, mas tem um risco: **arquivo que funciona não é o mesmo que arquivo que a gente entende**. A ideia aqui é fechar essa lacuna.

> **Regra deste repo:** não copiamos os arquivos originais. Explicamos os **conceitos** por trás deles, com exemplos mínimos e nomes genéricos (`minha-api`, `meu-banco`...). Para ver o arquivo real, abra o `app-api` lado a lado.

---

## Como estudar

1. Leia a **visão geral** primeiro — ela mostra o "mapa" de como tudo se conecta.
2. Depois siga os capítulos na ordem. Cada um responde três perguntas:
   - **O que é?** (a ideia, com uma analogia)
   - **Como aparece no app-api?** (o que o arquivo real está fazendo)
   - **Pontos de atenção** (o que olhar com desconfiança)
3. Leia o capítulo **"O que a IA deixou passar"** — é onde está o maior aprendizado.
4. Só então vá para a pasta **`02-evoluindo/`**, que mostra o próximo passo de cada conceito.

---

## Parte 1 — Entendendo o que existe (`01-entendendo/`)

| # | Capítulo | Arquivo(s) do app-api relacionados |
|---|---|---|
| 00 | [Visão geral: o mapa](01-entendendo/00-visao-geral.md) | todos |
| 01 | [Namespace](01-entendendo/01-namespace.md) | `k8s/namespace.yaml` |
| 02 | [Deployment e Pod](01-entendendo/02-deployment.md) | `k8s/deployment.yaml` |
| 03 | [Service](01-entendendo/03-service.md) | `k8s/service.yaml`, `k8s/deployment.yaml` |
| 04 | [Ingress](01-entendendo/04-ingress.md) | `k8s/ingress.yaml` |
| 05 | [ConfigMap e Secret](01-entendendo/05-configmap-e-secret.md) | `k8s/configmap.yaml`, `k8s/secret.yaml` |
| 06 | [StatefulSet (Postgres e Redis)](01-entendendo/06-statefulset.md) | `k8s/postgresql-statefulset.yaml`, `k8s/redis-statefulset.yaml` |
| 07 | [HPA — escala automática](01-entendendo/07-hpa.md) | `k8s/hpa.yaml` |
| 08 | [NetworkPolicy — firewall](01-entendendo/08-networkpolicy.md) | `k8s/networkpolicy.yaml` |
| 09 | [PrometheusRule — alertas](01-entendendo/09-alertas.md) | `k8s/alert-rules.yaml` |
| 10 | [Dockerfile](01-entendendo/10-dockerfile.md) | `Dockerfile` |
| 11 | [Docker Compose e infra local](01-entendendo/11-docker-compose.md) | `docker-compose.yml`, `infra-local.sh` |
| 12 | [Pipelines (GitHub Actions)](01-entendendo/12-pipelines.md) | `.github/workflows/*` |
| 13 | [Observabilidade com OpenTelemetry](01-entendendo/13-observabilidade.md) | `otel-collector-config.yaml`, runbook |
| 14 | [**O que a IA deixou passar**](01-entendendo/14-o-que-a-ia-deixou-passar.md) | vários |
| — | [Glossário](01-entendendo/glossario.md) | — |

## Parte 2 — Evoluindo os conceitos (`02-evoluindo/`)

Comece pelo [roteiro de evolução](02-evoluindo/README.md). Ele organiza as melhorias em níveis, do "conserto rápido" até "padrão de produção".

---

## Parte 3 — Agente para simular produção localmente

O arquivo [`.claude/agents/ambiente-local-prod.md`](.claude/agents/ambiente-local-prod.md) é um **agente do Claude Code**. Ele usa este material para montar um cluster k3d local que se comporta como produção:

- 3 nós com "zonas" simuladas, registry próprio, TLS e Ingress;
- Prometheus, Grafana, Alertmanager e coletor OpenTelemetry;
- manifestos novos em `deploy/local-prod/` (Kustomize), com todas as correções do capítulo 14 — sem mexer nos arquivos originais;
- uma bateria de testes de comportamento: rollout e rollback, readiness com o banco fora, PDB, NetworkPolicy, HPA, alertas, backup.

**Como usar:**

```bash
# 1. Copie o agente para o repositório do app-api (ou para ~/.claude/agents/ para usar em qualquer projeto)
mkdir -p ../app-api/.claude/agents
cp .claude/agents/ambiente-local-prod.md ../app-api/.claude/agents/

# 2. Abra o Claude Code no app-api e peça:
cd ../app-api && claude
> use o agente ambiente-local-prod para subir o ambiente
```

Outros pedidos que ele entende: "valide o ambiente local", "atualize o ambiente com a versão atual", "derrube o ambiente local".

> Requisitos: Docker (4 CPUs / 6 GB livres recomendados), k3d, kubectl, helm e git. O agente verifica e pede confirmação antes de instalar qualquer coisa ou apagar dados.

---

## Como testar os conceitos na prática

O `app-api` roda num cluster local **k3d** (Kubernetes leve dentro do Docker). Para brincar:

```bash
k3d cluster create estudo          # cria um cluster local
kubectl get nodes                  # confirma que está de pé
kubectl apply -f exemplo.yaml      # aplica um manifesto
kubectl get all -n meu-namespace   # vê o que foi criado
kubectl describe pod <nome>        # o comando que mais ensina!
kubectl delete -f exemplo.yaml     # limpa
```

> **Dica de ouro:** sempre que não entender um campo, rode `kubectl explain deployment.spec.template.spec.containers.livenessProbe` — o próprio cluster explica cada campo.
