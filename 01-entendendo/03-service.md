# 03 — Service

## O que é?

Pods mudam de IP toda vez que renascem. Um **Service** dá um **endereço fixo** (IP e nome DNS) que sempre aponta para os pods certos, e ainda **divide o tráfego** entre eles.

> **Analogia:** o telefone fixo de uma loja. Os atendentes mudam, o número continua o mesmo, e a central distribui as ligações.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-api
spec:
  type: ClusterIP
  selector:
    app: minha-api      # manda para pods com esta label
  ports:
    - port: 80          # porta do Service
      targetPort: 8000  # porta do container
```

## As duas portas

- `port`: onde **quem chama** bate (`http://minha-api:80`).
- `targetPort`: onde o **container** escuta.

No app-api as duas são iguais (8000), o que simplifica, mas esconde que são conceitos diferentes.

## Tipos de Service

| Tipo | Quem consegue acessar | Quando usar |
|---|---|---|
| `ClusterIP` (padrão) | Só quem está dentro do cluster | Comunicação interna (API → banco) |
| `NodePort` | Qualquer um, pela porta alta do nó (30000–32767) | Testes rápidos |
| `LoadBalancer` | A internet, via balanceador da nuvem | Exposição direta em nuvem |
| *Headless* (`clusterIP: None`) | Sem IP próprio; o DNS devolve o IP de cada pod | StatefulSets (cap. 06) |

O app-api usa só `ClusterIP` e expõe para fora via **Ingress** (cap. 04) — o jeito mais comum.

## Como aparece no app-api

Existem **três** Services: um para a API, um para o Postgres e um para o Redis. O nome do Service vira o "endereço" usado nas configurações (por isso a config de conexão com o banco aponta para o nome do Service do Postgres, não para um IP).

⚠️ Na verdade a API tem **dois** Services apontando para os mesmos pods, com nomes diferentes — um em `service.yaml` e outro escondido no final de `deployment.yaml`. Só um deles é usado pelo Ingress. Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## Dica: o `---`

Um arquivo YAML pode conter **vários** recursos separados por uma linha com `---`. É prático, mas facilita esquecer que algo está lá — foi exatamente assim que o Service duplicado passou despercebido.

## Comandos para explorar

```bash
kubectl get svc -n meu-projeto
kubectl get endpoints minha-api -n meu-projeto   # quais IPs de pod estão "atrás" do Service
kubectl port-forward svc/minha-api 8000:80        # acessa do seu computador
```

> Se `get endpoints` vier vazio, o seletor não bateu com nenhuma label ou nenhum pod está *ready*.

Próximo: [04 — Ingress](04-ingress.md)
