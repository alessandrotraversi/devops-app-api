# 14 — O que a IA deixou passar

Configurações geradas por IA costumam ter um padrão: **cada arquivo isolado parece correto**, mas **as conexões entre eles** têm falhas. A IA escreve um arquivo de cada vez, sem verificar se o nome usado aqui é o mesmo declarado ali.

Este capítulo lista o que foi encontrado no app-api, por **conceito**. Cada item tem: o que está acontecendo, por que importa e onde aprender a corrigir.

> Não é uma crítica: é exatamente esse tipo de revisão que transforma "copiei da IA" em "eu entendo o que está rodando".

Legenda de gravidade: 🔴 quebra ou expõe algo · 🟡 funciona por sorte / risco futuro · 🔵 limpeza

---

## A. Conexões entre arquivos

### 🔴 A1. `latest` + `IfNotPresent` = deploy que não atualiza
O pipeline gera uma imagem nova sempre com a **mesma tag** (`latest`) e o Deployment também aponta para `latest`. Quando você roda `kubectl apply`, o Kubernetes compara o YAML novo com o antigo: **nada mudou** (mesma string de imagem) → **não há rollout**. Os pods antigos continuam rodando o código antigo. E mesmo se um pod reiniciar, `IfNotPresent` pode reaproveitar uma imagem antiga já presente no nó.

**Sintoma típico:** "fiz deploy, o pipeline ficou verde, mas a mudança não aparece."
→ [02-evoluindo/02 — Imagens e versões](../02-evoluindo/02-imagens-e-versoes.md)

### 🟡 A2. Dois Services para a mesma API
Um Service está em `service.yaml` e outro escondido depois de um `---` no `deployment.yaml`, com outro nome. O Ingress e o smoke test usam um deles; o outro é peso morto. Funciona, mas confunde quem for depurar.

### 🟡 A3. Readiness usa o endpoint errado
A aplicação tem `/health` (só diz "estou vivo") e `/ready` (testa o banco). As duas probes usam `/health`. Resultado: se o banco cair, os pods continuam recebendo tráfego e devolvendo erro, em vez de saírem do balanceamento.
→ [02-evoluindo/07 — Resiliência](../02-evoluindo/07-resiliencia.md)

### 🟡 A4. StatefulSet apontando para Service não-headless
O `serviceName` dos StatefulSets aponta para Services comuns. Com 1 réplica, ninguém percebe. Ao escalar o banco, os nomes por pod não existem.

### 🟡 A5. Probe do Postgres com usuário "chumbado"
O usuário vem do Secret, mas o comando da probe tem o nome escrito à mão. Se trocar o usuário no Secret, a probe quebra e o banco entra em loop de reinício.

---

## B. Segredos

### 🔴 B1. Secret commitado no Git
Base64 não é criptografia (cap. 05). A senha do banco está pública no repositório.

### 🔴 B2. Senha também no ConfigMap
A URL de conexão completa, com usuário e senha, está no **ConfigMap**. Mesmo que o Secret fosse protegido, o vazamento continuaria.
→ [02-evoluindo/01 — Segredos](../02-evoluindo/01-segredos.md)

### 🟡 B3. Redis sem senha
Combinado com a NetworkPolicy permissiva (C1), qualquer pod do cluster pode ler e apagar o cache.

---

## C. Segurança de rede e de execução

### 🟡 C1. NetworkPolicy que "parece" proteger
Libera entrada de qualquer namespace e saída para qualquer lugar do cluster. Existe, mas protege muito pouco. E não há política alguma para o Postgres e o Redis.
→ [02-evoluindo/05 — Segurança](../02-evoluindo/05-seguranca.md)

### 🔵 C2. `securityContext` incompleto
A API já está bem (não-root, disco somente-leitura), mas faltam `allowPrivilegeEscalation: false`, remover *capabilities* e perfil `seccomp`. Postgres e Redis não têm nenhum.

---

## D. Autoscaling

### 🟡 D1. Métrica do HPA que não existe
A segunda métrica (`type: Pods`, nome `cpu`) exige um adaptador de métricas customizadas que não está instalado. O HPA fica registrando erro. A primeira métrica sozinha já faz o trabalho.

---

## E. Observabilidade

### 🔴 E1. Alertas sem dados
Os alertas de erro e latência consultam métricas HTTP no formato do Prometheus, mas:
- o endpoint `/metrics` da aplicação devolve um **JSON de status**, não métricas;
- não há `ServiceMonitor` dizendo ao Prometheus onde buscar;
- os nomes das métricas no runbook e nos alertas são diferentes entre si.

Resultado: esses dois alertas **nunca disparam** — o pior tipo de alerta, porque dá falsa sensação de segurança.

### 🟡 E2. Alertas não seguem os SLOs
O runbook diz "erro < 0,1%" e "P95 < 200 ms". Os alertas disparam com erro > 5% e P95 > 2 s — limites de 10 a 50 vezes mais frouxos. O SLO pode ser violado por dias sem nenhum alerta.

### 🔵 E3. Imagem com ambiente fixo
A imagem já sai dizendo que é `dev` e para onde mandar telemetria. Em outro ambiente, os dados chegam com o rótulo errado.
→ [02-evoluindo/08 — Observabilidade](../02-evoluindo/08-observabilidade.md)

---

## F. Pipeline

### 🔴 F1. Gates que não reprovam
`|| true` em lint, testes, Trivy, espera de rollout e smoke test; quality gate do Sonar só gera aviso. Um PR que quebra todos os testes **é deployado igual**.

### 🟡 F2. Deploy duplicado
Os passos do deploy automático e do manual são cópias. Divergem com o tempo.

### 🟡 F3. "Destroy" que não destrói tudo
`kubectl delete all` deixa para trás ConfigMap, Secret, PVC (com os dados do banco), Ingress, NetworkPolicy...

### 🟡 F4. Ordem de aplicação
`kubectl apply -f k8s/` aplica em ordem alfabética; o namespace não vem primeiro. Num cluster novo, a primeira execução pode falhar.
→ [02-evoluindo/09 — Pipeline maduro](../02-evoluindo/09-pipeline-maduro.md)

---

## G. Pequenas inconsistências

- 🔵 README fala em Postgres 16; os manifestos usam 15.
- 🔵 `curl` instalado no estágio de build sem uso, e na imagem final só para um HEALTHCHECK que o Kubernetes ignora.
- 🔵 IP fixo do coletor no Compose.
- 🔵 Falta `.dockerignore`.

---

## Lição geral: como revisar config gerada por IA

1. **Siga os nomes.** Para cada nome (Service, label, Secret, métrica), procure onde ele é declarado e onde é usado. `grep` é seu amigo.
2. **Pergunte "e se mudar?"** E se a tag mudar? E se a senha mudar? E se o banco cair?
3. **Desconfie de `|| true`, `latest` e seletores vazios `{}`.**
4. **Teste o caminho triste.** Derrube o banco e veja se a readiness tira os pods. Gere erro 500 e veja se o alerta dispara.
5. **Rode ferramentas que leem o conjunto:** `kubectl apply --dry-run=server`, `kubeconform`, `trivy config`, `kube-linter`.
