# 02 — Deployment e Pod

Este é o arquivo mais importante. Vamos por partes.

## O que é?

- **Pod**: a menor unidade que o Kubernetes roda. Normalmente = 1 container (o seu app). Pods são **descartáveis**: morrem e nascem com outro nome e outro IP.
- **Deployment**: o "gerente" dos pods. Você diz quantas cópias quer e qual imagem usar; ele garante isso e cuida das atualizações.

> **Analogia:** o Pod é um funcionário temporário; o Deployment é o RH que garante que sempre haja N funcionários no turno — e que troca a equipe aos poucos quando muda o uniforme (a versão).

Por baixo, o Deployment cria um **ReplicaSet**, que cria os Pods. Você quase nunca mexe no ReplicaSet diretamente.

```
Deployment ──cria──► ReplicaSet ──cria──► Pod, Pod, Pod
```

## Anatomia, bloco por bloco

Vamos usar um esqueleto mínimo e explicar cada parte. O app-api usa exatamente esses blocos.

### 1. Réplicas e seletor

```yaml
spec:
  replicas: 2
  selector:
    matchLabels:
      app: minha-api
  template:
    metadata:
      labels:
        app: minha-api
```

- `replicas`: quantas cópias do pod. Com 2, se uma cair, a outra segura.
- `selector`: **como o Deployment reconhece os pods que são dele**.
- `template`: o "molde" do pod. As labels do molde **precisam** bater com o seletor.

No app-api: 2 réplicas, e os pods recebem também uma label `tier: backend` (só informativa — ninguém seleciona por ela).

### 2. Contexto de segurança (`securityContext`)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

Por padrão, muitos containers rodam como **root**. Se alguém invadir a aplicação, ganha poderes de root dentro do container. Aqui o app-api:

- **Proíbe rodar como root** (no nível do pod).
- **Força o usuário de ID 1000** — o mesmo criado no Dockerfile.
- **Deixa o sistema de arquivos somente-leitura** (`readOnlyRootFilesystem`, no nível do container). Um invasor não consegue gravar arquivos maliciosos.

Consequência do "somente leitura": a aplicação às vezes **precisa** gravar algo temporário. Por isso existem os volumes `/tmp` e `/run` — veja o bloco 6.

### 3. Imagem

```yaml
image: minha-api:1.4.2
imagePullPolicy: IfNotPresent
```

- `image`: qual pacote Docker rodar.
- `imagePullPolicy: IfNotPresent`: "só baixe a imagem se ela ainda não existir no nó".

⚠️ No app-api a imagem usa a tag `latest` junto com `IfNotPresent`. Essa combinação é traiçoeira — explicada em detalhe no [capítulo 14](14-o-que-a-ia-deixou-passar.md).

### 4. Variáveis de ambiente vindas de fora

```yaml
envFrom:
  - configMapRef:
      name: config-da-api
  - secretRef:
      name: segredos-da-api
```

Em vez de escrever as configurações dentro do Deployment, ele **puxa todas as chaves** de um ConfigMap e de um Secret e as transforma em variáveis de ambiente. Assim, a mesma imagem roda em qualquer ambiente, mudando só a configuração. (Capítulo [05](05-configmap-e-secret.md).)

### 5. Recursos: `requests` e `limits`

```yaml
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 500m, memory: 512Mi }
```

| Campo | Significa | Usado por |
|---|---|---|
| `requests` | "Eu **preciso** de pelo menos isso" | O **agendador** (escolhe em qual nó cabe o pod) e o **HPA** (calcula a % de uso) |
| `limits` | "Eu **nunca** posso passar disso" | O nó, em tempo de execução |

Unidades:
- `100m` de CPU = 100 **mili**cores = 10% de um núcleo.
- `128Mi` = 128 mebibytes de memória.

O que acontece ao passar do limite:
- **CPU** → o container fica **lento** (é "estrangulado"), mas não morre.
- **Memória** → o container é **morto** (`OOMKilled`) e reiniciado. Há até um alerta para isso no app-api.

### 6. Volumes temporários (`emptyDir`)

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

Um `emptyDir` é uma pasta vazia que nasce com o pod e **morre com ele**. Serve como "rascunho". O app-api monta dois (`/tmp` e `/run`) justamente para compensar o sistema de arquivos somente-leitura.

### 7. Probes: o Kubernetes perguntando "tudo bem aí?"

Esta é a parte mais mal entendida. Existem 3 tipos de probe:

| Probe | Pergunta | Se falhar... |
|---|---|---|
| **liveness** | "Você está **vivo**?" | O container é **reiniciado** |
| **readiness** | "Você está **pronto** para receber tráfego?" | O pod sai do Service (para de receber requisições), mas **não** reinicia |
| **startup** | "Você já **terminou de ligar**?" | Enquanto não passar, as outras duas nem rodam |

Parâmetros comuns:

| Parâmetro | Significado |
|---|---|
| `initialDelaySeconds` | Espera X segundos antes da primeira pergunta |
| `periodSeconds` | Pergunta a cada X segundos |
| `timeoutSeconds` | Quanto tempo espera a resposta |
| `failureThreshold` | Quantas falhas seguidas até agir |

No app-api, liveness e readiness fazem uma chamada HTTP para o **mesmo** endpoint de saúde. A readiness pergunta com mais frequência (a cada 10 s) que a liveness (a cada 30 s) — faz sentido: tirar do tráfego é barato, reiniciar é caro.

⚠️ Mas usar o **mesmo** endpoint para as duas perde a graça da readiness — a aplicação tem um endpoint `/ready` que testa o banco e não está sendo usado. Detalhes no [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## O que o Deployment faz numa atualização (rolling update)

Quando você muda a imagem, o Deployment **não** derruba tudo de uma vez. Ele sobe um pod novo, espera ficar *ready*, derruba um antigo, e repete. Por isso a readiness probe é tão importante: é ela que diz "pode mandar tráfego para o novo".

O app-api não define `strategy`, então usa o padrão (até 25% a mais de pods durante a troca e até 25% indisponíveis).

## Comandos para explorar

```bash
kubectl get deploy,rs,pods -n meu-projeto
kubectl describe pod <pod>            # veja eventos, probes e motivo de reinícios
kubectl rollout status deploy/<nome>  # acompanha uma atualização
kubectl rollout history deploy/<nome>
kubectl rollout undo deploy/<nome>    # volta para a versão anterior
kubectl top pods                      # uso real de CPU/memória (precisa do metrics-server)
```

Próximo: [03 — Service](03-service.md)
