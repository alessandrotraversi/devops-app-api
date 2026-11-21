# 07 — HPA (Horizontal Pod Autoscaler)

## O que é?

O **HPA** aumenta ou diminui o número de réplicas de um Deployment com base em uma métrica (geralmente CPU).

- **Horizontal** = mais cópias (escala "para o lado").
- **Vertical** = cópias maiores (mais CPU/memória por pod) — isso é outro recurso, o VPA.

> **Analogia:** um supermercado que abre mais caixas quando a fila cresce e fecha quando esvazia.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## A conta que o HPA faz

```
réplicas desejadas = arredondar_para_cima( réplicas atuais × uso atual / alvo )
```

Exemplo: 2 pods usando em média 140% do `request` de CPU, alvo de 70% → 2 × 140/70 = **4 pods**.

⚠️ **"Utilization" é em relação ao `request`, não ao `limit`.** Se o pod pede 100m e usa 70m, está a 70%. Por isso, sem `requests` definidos, o HPA não funciona.

## Como aparece no app-api

- Mínimo 2, máximo 10 réplicas.
- Alvo: 70% de uso de CPU (relativo ao request).
- Uma segunda métrica do tipo `Pods` com valor médio de CPU em milicores.
- Bloco `behavior` com janelas de estabilização:
  - **Subir**: olha o último minuto antes de decidir.
  - **Descer**: olha os últimos 5 minutos antes de decidir.

### Por que descer é mais lento que subir?

Para evitar "efeito sanfona" (*flapping*): um pico rápido de tráfego faria subir, cair, subir... Subir rápido protege o usuário; descer devagar evita desperdício de idas e vindas.

### Por que `minReplicas: 2` e o Deployment também diz 2?

Quando existe HPA, **ele manda** no número de réplicas. O `replicas` do Deployment só vale no primeiro momento. Na prática, se você reaplicar o Deployment, ele "reseta" para 2 e o HPA corrige de novo logo depois.

⚠️ A segunda métrica (tipo `Pods`, chamada `cpu`) **não é uma métrica padrão**: o tipo `Pods` espera métricas customizadas, que precisam de um adaptador extra (ex.: Prometheus Adapter). Sem ele, o HPA fica reclamando que não consegue lê-la. E ela é redundante com a primeira. Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## Pré-requisito: metrics-server

O HPA precisa saber o uso de CPU dos pods. Quem informa isso é o **metrics-server** (o k3s já traz).

## Comandos para explorar

```bash
kubectl get hpa -n meu-projeto -w          # -w = acompanha ao vivo
kubectl describe hpa minha-api             # mostra eventos e erros de métrica
# Gerar carga para ver escalar:
kubectl run carga --rm -it --image=busybox -- sh -c "while true; do wget -q -O- http://minha-api/health; done"
```

Próximo: [08 — NetworkPolicy](08-networkpolicy.md)
