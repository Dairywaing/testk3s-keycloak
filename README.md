
**Тестовый стенд в wsl(ubuntu24.4):**

- Keycloak-operator

- PostgreSQL-cnpg

- Бэкапы в S3 через  Barman Cloud plugin

- TLS от **cert-manager**, сертификаты самоподписанные.  Хостнеймы в `*.test.local`

- ArgoCD - для деплоя
## 1. Установка k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

Для корректной работы необходимо скопировать kubeconfig:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
export KUBECONFIG="$HOME/.kube/config"
kubectl get nodes ## для проверки что все окей
```

## 2. Создать записи в /etc/hosts на windows для дальнейшей проверки

```
## ip = адрес wsl машины, а не 127.0.0.1
127.0.0.1 sso.test.local
127.0.0.1 argocd.test.local
127.0.0.1 s3.test.local
127.0.0.1 app.test.local
```

## 3. Склонировать репозиторий

```bash
git clone https://github.com/Dairywaing/testk3s-keycloak.git
```

## 4. Секреты

```bash
# создаем namespaces
kubectl create ns argocd cert-manager cnpg-system keycloak s3

# генерируем секереты
kubectl -n s3 create secret generic rustfs-credentials \
  --from-literal=access-key=changeme! --from-literal=secret-key=changeme!
kubectl -n s3 create secret generic keycloak-db-s3 \
  --from-literal=access-key=changeme! --from-literal=secret-key=changeme!
kubectl -n keycloak create secret generic rustfs-credentials \
  --from-literal=access-key=changeme! --from-literal=secret-key=changeme!
kubectl -n keycloak create secret generic keycloak-db-s3 \
  --from-literal=access-key=changeme! --from-literal=secret-key=changeme!

# postgres
kubectl -n keycloak create secret generic keycloak-db-app \
  --from-literal=username=keycloak --from-literal=password=keycloak \
  --from-literal=dbname=keycloak

# kc admin user 
kubectl -n keycloak create secret generic keycloak-bootstrap-admin \
  --from-literal=username=admin --from-literal=password=keycloak
```
## 5. Argo CD

```bash
## установка argocd в кластер
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash

kubectl apply --server-side --force-conflicts -f argocd/namespace.yaml
sed -e 's|${ARGOCD_HOSTNAME}|argocd.test.local|g' \
    -e 's|${ACME_ISSUER}|selfsigned-bootstrap|g' \
  argocd/values.yaml > /tmp/argocd-values.yaml
  
helm repo add argo https://argoproj.github.io/argo-helm --force-update
helm upgrade --install argocd argo/argo-cd -n argocd \
   --values /tmp/argocd-values.yaml
```

Без этого Secret Argo CD не скачает OCI-чарт Keycloak-оператора:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: repo-keycloak-operator-oci
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  name: keycloak-operator-oci
  type: helm
  url: quay.io/keycloak
  enableOCI: "true"
EOF
```

Root Application - добавляем репо в argocd:

```bash
sed -e 's|${GITOPS_REPO_URL}|https://github.com/Dairywaing/testk3s-keycloak.git|g' \
    -e 's|${GITOPS_REPO_REVISION}|main|g' \
  argocd/root-app.yaml | kubectl apply --server-side --force-conflicts -f -

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

## 6. Проверка

```bash
# ~ 5 минут
kubectl -n argocd get applications -w

kubectl -n keycloak get cluster keycloak-db \
  -o jsonpath='{.status.phase}{"  primary="}{.status.currentPrimary}{"\n"}'
kubectl -n keycloak get keycloak keycloak \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'; echo
kubectl -n s3 get job rustfs-create-bucket
kubectl -n keycloak get backup
```

| Что      | Адрес                     |
| -------- | ------------------------- |
| Keycloak | https://sso.test.local    |
| Argo CD  | https://argocd.test.local |
| RustFS   | https://s3.test.local     |
