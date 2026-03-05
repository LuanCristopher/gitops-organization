# GitOps Boilerplate - ArgoCD & Kustomize

Este repositório serve como um **Template Padrão** para implantação de fluxo GitOps utilizando **ArgoCD** e **Kustomize**. 

O objetivo é padronizar e acelerar a entrega de infraestrutura e aplicações para novos clientes, separando configurações globais (`base`) de configurações específicas de ambiente (`overlays`).

---

## Pré-requisitos
* Cluster Kubernetes rodando (GKE, AKS, EKS, etc.).
* `kubectl` instalado e configurado com o contexto do cluster do cliente.
* Terminal Bash (Linux/Mac ou WSL no Windows).

---

## Passo 1: Instalar o ArgoCD no Cluster

Antes de gerar os arquivos, precisamos garantir que o ArgoCD está rodando no cluster do cliente. Execute os comandos abaixo no seu terminal para criar o namespace e instalar a versão estável:

```bash
# 1. Cria o namespace dedicado para o ArgoCD
kubectl create namespace argocd

# 2. Aplica os manifestos oficiais de instalação do ArgoCD
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

# 3. Acompanhe a subida dos pods (aguarde todos ficarem com status 'Running')
kubectl get pods -n argocd -w

```

```bash
Passo 2: Criar a Estrutura de Diretórios (Script 1)
Clone este repositório vazio na máquina, abra o terminal na raiz do repositório e rode o script abaixo para montar a árvore de pastas e arquivos vazios.

#!/bin/bash
echo "Criando a estrutura de diretórios Kustomize..."

mkdir -p base/{namespace,secrets,configmaps,rbac,common,ingress-controller}
touch base/namespace/app-namespace.yaml
touch base/secrets/external-secrets.yaml
touch base/configmaps/common-configmap.yaml
touch base/rbac/app-role.yaml
touch base/common/argocd-ingress.yaml
touch base/ingress-controller/kong.yaml

for env in dev staging production; do
  mkdir -p overlays/$env/{app1,app2,app3}
  for app in app1 app2 app3; do
    touch overlays/$env/$app/values.yaml
    touch overlays/$env/$app/kustomization.yaml
  done
done

mkdir -p apps/{app1,app2,app3}
for app in app1 app2 app3; do
  touch apps/$app/deployment.yaml
  touch apps/$app/service.yaml
  touch apps/$app/ingress.yaml
  touch apps/$app/kustomization.yaml
done
echo "✅ Estrutura criada com sucesso!"

```


```bash
Passo 3: Preencher os Manifestos (Script 2)
Rode o script abaixo para injetar o código YAML dentro dos arquivos recém-criados.
Atenção: Após rodar este script, faça uma busca no seu editor pelas variáveis como {{NOME_DO_CLIENTE}}, {{URL_DO_REPO}} etc., e substitua pelos dados reais do projeto do cliente.

#!/bin/bash
echo "Preenchendo os arquivos do Template GitOps..."

cat << 'EOF' > base/namespace/app-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: "{{NAMESPACE_APPS}}"
  labels:
    environment: global
EOF

cat << 'EOF' > base/secrets/external-secrets.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: "{{NAMESPACE_APPS}}"
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: secret-store
    kind: ClusterSecretStore
  target:
    name: app-secrets-k8s
  data:
  - secretKey: db-password
    remoteRef:
      key: "{{CAMINHO_DO_SECRET_NO_CLOUD}}"
EOF

cat << 'EOF' > base/configmaps/common-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: common-config
  namespace: "{{NAMESPACE_APPS}}"
data:
  COMPANY_NAME: "{{NOME_DO_CLIENTE}}"
  ENVIRONMENT: "global"
EOF

cat << 'EOF' > base/rbac/app-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: "{{NAMESPACE_APPS}}"
  name: app-developer-role
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["pods", "deployments", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
EOF

cat << 'EOF' > base/common/argocd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: "argocd.{{DOMINIO_DO_CLIENTE}}.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
EOF

cat << 'EOF' > base/ingress-controller/kong.yaml
apiVersion: [configuration.konghq.com/v1](https://configuration.konghq.com/v1)
kind: KongPlugin
metadata:
  name: rate-limit-global
  namespace: "{{NAMESPACE_APPS}}"
config:
  minute: 100
  limit_by: ip
  policy: local
plugin: rate-limiting
EOF

for app in app1 app2 app3; do
cat << EOF > apps/$app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${app}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${app}
  template:
    metadata:
      labels:
        app: ${app}
    spec:
      containers:
      - name: ${app}
        image: "{{REGISTRY_URL}}/${app}:latest"
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
EOF

cat << EOF > apps/$app/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ${app}-svc
spec:
  selector:
    app: ${app}
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
EOF

cat << EOF > apps/$app/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${app}-ingress
spec:
  rules:
  - host: "${app}.{{DOMINIO_DO_CLIENTE}}.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${app}-svc
            port:
              number: 80
EOF

cat << 'EOF' > apps/$app/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
EOF
done

declare -A replicas=( ["dev"]="1" ["staging"]="2" ["production"]="4" )

for env in dev staging production; do
  for app in app1 app2 app3; do
    
cat << EOF > overlays/$env/$app/values.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${app}
spec:
  replicas: ${replicas[$env]}
  template:
    spec:
      containers:
      - name: ${app}
        image: "{{REGISTRY_URL}}/${app}:{{TAG_DA_IMAGEM_$env}}"
EOF

cat << EOF > overlays/$env/$app/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../apps/${app}
patches:
  - path: values.yaml
namespace: "{{NAMESPACE_APPS}}-${env}"
EOF

  done
done
echo "✅ Arquivos Kustomize preenchidos com sucesso!"

```

Após rodar o script e ajustar as variáveis, faça o commit e push para o GitHub:

```bash
git add .
git commit -m "feat: setup inicial do kustomize template"
git push origin main

```

```bash
Passo 4: Conectar o ArgoCD ao Repositório (Application CRD)
Agora que os arquivos estão no GitHub, precisamos dizer ao ArgoCD para olhar para a nossa pasta overlays/production (ou dev/staging) e aplicar as mudanças no cluster.

Crie um arquivo chamado argocd-app-production.yaml na sua máquina local com o conteúdo abaixo:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: '[https://github.com/](https://github.com/){{SUA_ORG}}/{{SEU_REPO}}.git' # <-- Altere para o link do repositório
    targetRevision: main
    path: overlays/production # <-- Aponta diretamente para o ambiente desejado
  destination:
    server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)' # O próprio cluster onde o Argo está rodando
    namespace: '{{NAMESPACE_APPS}}-production' # O namespace destino (deve existir ou ser criado pelo Argo)
  syncPolicy:
    automated:
      prune: true      # Remove recursos que foram deletados no git
      selfHeal: true   # Corrige se alguém alterar algo manualmente no cluster
    syncOptions:
      - CreateNamespace=true # Cria o namespace automaticamente se não existir

```

Aplique esse arquivo no cluster:
```bash
kubectl apply -f argocd-app-production.yaml
```

