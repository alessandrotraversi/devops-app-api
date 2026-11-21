# Glossário

| Termo | Explicação curta |
|---|---|
| **Annotation** | Metadado livre, lido por ferramentas específicas (ex.: Ingress Controller) |
| **Base64** | Forma de *codificar* texto. Não é criptografia |
| **CI/CD** | Integração contínua (testar sempre) / entrega contínua (publicar sempre) |
| **Cluster** | Conjunto de máquinas (nós) gerenciadas pelo Kubernetes |
| **ConfigMap** | Configurações não sensíveis em chave/valor |
| **Container** | Processo isolado rodando a partir de uma imagem |
| **CRD** | *Custom Resource Definition*: tipo novo adicionado ao Kubernetes (ex.: PrometheusRule) |
| **CrashLoopBackOff** | Container que inicia e morre repetidamente; o Kubernetes espera cada vez mais entre tentativas |
| **Deployment** | Gerencia réplicas de pods sem estado e suas atualizações |
| **Egress / Ingress (rede)** | Tráfego de saída / de entrada |
| **emptyDir** | Volume temporário que vive enquanto o pod vive |
| **Error budget** | Quanto de falha o SLO permite |
| **Headless Service** | Service sem IP próprio, que dá um DNS por pod |
| **HPA** | Escala o número de pods automaticamente |
| **Imagem** | Pacote imutável com sistema + dependências + código |
| **Ingress (recurso)** | Regras de roteamento HTTP de fora para dentro do cluster |
| **Ingress Controller** | Programa que implementa as regras do Ingress (Traefik, NGINX) |
| **k3d / k3s** | k3s = Kubernetes leve; k3d = k3s rodando dentro do Docker |
| **Label** | Etiqueta chave/valor usada para *selecionar* recursos |
| **Liveness probe** | Verifica se o container está vivo; falhou → reinicia |
| **Namespace** | "Pasta" lógica dentro do cluster |
| **NetworkPolicy** | Firewall entre pods |
| **OOMKilled** | Container morto por exceder o limite de memória |
| **OTLP** | Protocolo do OpenTelemetry para enviar telemetria |
| **P95** | Valor abaixo do qual estão 95% das medições |
| **Pod** | Menor unidade executável do Kubernetes (1+ containers) |
| **PromQL** | Linguagem de consulta do Prometheus |
| **PVC / PV** | Pedido de disco / disco de fato |
| **Readiness probe** | Verifica se o pod pode receber tráfego; falhou → sai do Service |
| **ReplicaSet** | Mantém N cópias de um pod (criado pelo Deployment) |
| **Rolling update** | Troca de versão aos poucos, sem derrubar tudo |
| **Runbook** | Manual de diagnóstico e ação para incidentes |
| **Runner self-hosted** | Máquina própria que executa os pipelines |
| **Secret** | Dados sensíveis (base64; proteger com ferramentas extras) |
| **Service** | Endereço estável que balanceia tráfego entre pods |
| **SLI / SLO / SLA** | Indicador / objetivo / acordo contratual de nível de serviço |
| **StatefulSet** | Gerencia pods com identidade e disco estáveis (bancos) |
| **StorageClass** | "Fornecedor" que cria discos sob demanda |
| **Tag (imagem)** | Rótulo de versão de uma imagem (`:1.2.3`, `:latest`) |
