# Evoluindo os conceitos

Esta pasta mostra **o próximo passo** de cada conceito visto em `01-entendendo/`. Não é uma lista de "jeito certo" — é uma escada. Suba um degrau por vez e só suba quando o anterior fizer sentido.

## Os três níveis

| Nível | Objetivo | Esforço |
|---|---|---|
| 🥉 **1 — Consertar** | Fazer o que existe funcionar como se imagina | Horas |
| 🥈 **2 — Profissionalizar** | Padrões que qualquer time usa | Dias |
| 🥇 **3 — Produção** | Seguro, automatizado, auditável | Semanas |

## Roteiro sugerido

| Ordem | Tema | Nível 1 | Nível 2 | Nível 3 |
|---|---|---|---|---|
| 1 | [Segredos](01-segredos.md) | Tirar senha do ConfigMap e do Git | Sealed Secrets / SOPS | External Secrets + cofre |
| 2 | [Imagens e versões](02-imagens-e-versoes.md) | Tag por commit | Registry + `.dockerignore` | Digest, assinatura, SBOM |
| 3 | [Empacotando manifestos](03-empacotando-manifestos.md) | Ordem de aplicação | Kustomize (base + ambientes) | Helm chart |
| 4 | [GitOps](04-gitops.md) | — | Argo CD lendo o repo | Promoção entre ambientes |
| 5 | [Segurança](05-seguranca.md) | securityContext completo | NetworkPolicy *default deny* | Pod Security, RBAC, TLS |
| 6 | [Banco de dados](06-banco-de-dados.md) | Headless Service, probe sem nome fixo | Backup | Operator ou banco gerenciado |
| 7 | [Resiliência](07-resiliencia.md) | Readiness no `/ready` | PDB, startupProbe, estratégia | Spread entre zonas |
| 8 | [Observabilidade](08-observabilidade.md) | Métricas reais + ServiceMonitor | Alertas = SLOs | Burn rate multi-janela, traces |
| 9 | [Pipeline maduro](09-pipeline-maduro.md) | Remover `\|\| true` | Um único workflow reutilizável | Ambientes, aprovação, rollback |

> **Se for fazer só três coisas:** tag de imagem por commit (02), tirar `|| true` dos testes (09) e tirar a senha do Git (01). Juntas, elas resolvem os problemas mais graves do capítulo 14.

## Como cada arquivo está organizado

1. **Onde estamos** — o estado atual em uma frase.
2. **Degraus** — nível 1, 2 e 3, cada um com o *porquê* e um exemplo mínimo.
3. **Como validar** — o teste que prova que funcionou.
