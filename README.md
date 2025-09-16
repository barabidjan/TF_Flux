# Terraform_flux
# Створення коду Terraform для Flux на kind_cluster

1. В `main.tf` змінимо модуль що відповідає за розгортання кластеру згідно з завданням. Але оберемо гілку модуля, що може працювати без створення файлу `kubeconfig` та використовую інший вид авторизації

```hcl
module "kind_cluster" {
  source = "github.com/den-vasyliev/tf-kind-cluster?ref=cert_auth"
}
```
2. Виконаємо ініціалізацію terraform:
```sh
✗ terraform init
Terraform has been successfully initialized!
```

- Виконуємо оновлення провайдера командою:
```sh
terraform init -upgrade
```
- Перевіримо які модулі були створені щоб видалити повністю файл стану:
```sh
✗ terraform state list
module.flux_bootstrap.flux_bootstrap_git.this
module.kind_cluster.kind_cluster.this
✗ terraform state rm module.flux_bootstrap.flux_bootstrap_git.this
Removed module.flux_bootstrap.flux_bootstrap_git.this
Successfully removed 1 resource instance(s).
✗ terraform state rm module.kind_cluster.kind_cluster.this        
Removed module.kind_cluster.kind_cluster.this
Successfully removed 1 resource instance(s).
✗ kind get clusters            
kind-cluster
✗ kind delete clusters kind-cluster
Deleted clusters: ["kind-cluster"]
```


3. Перевіримо код 
```sh
✗ tf validate
Success! The configuration is valid.
```

4. Виконаємо початкові команду `terraform apply`.
```sh
✗ tf apply
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

5. Створені ресурси:
```sh
$ tf state list
module.flux_bootstrap.flux_bootstrap_git.this
module.github_repository.github_repository.this
module.github_repository.github_repository_deploy_key.this
module.kind_cluster.kind_cluster.this
module.tls_private_key.tls_private_key.this
```

6. Розміщення файлу в [bucket](https://console.cloud.google.com/storage/browser)  
Щоб розмістити файл state в бакеті, ви можете використовувати команду terraform init з опцією --backend-config. Наприклад, щоб розмістити файл state в бакеті Google Cloud Storage, ви можете виконати наступну команду:
```sh
# Створимо bucket:
$ gsutil mb gs://inv-secret
Creating gs://inv-secret/...

# Перевірити вміст диску:
$ gsutil ls gs://inv-secret
gs://inv-secret/terraform/
```
7. Як створити bucket [читаємо документацію](https://developer.hashicorp.com/terraform/language/settings/backends/gcs#example-configuration) та додаємо до основного файлу конфігурації наступний код:

```hcl
terraform {
  backend "gcs" {
    bucket  = "tf-state-prod"
    prefix  = "inv-secret"
  }
}
```
```sh
$ terraform init
$ tf show | more
```

8. Перевіримо список ns по стан поду системи flux:
```sh
✗ k get ns
NAME                 STATUS   AGE
default              Active   16m
flux-system          Active   15m
kube-node-lease      Active   16m
kube-public          Active   16m
kube-system          Active   16m
local-path-storage   Active   16m

✗ ydibka@cloudshell:~/flux-gitops/clusters/demo (terraformtest-472215)$ k get po -n flux-system
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-b6767d66-9hv6t             1/1     Running   0          65m
kustomize-controller-57c7ff5596-rcggs      1/1     Running   0          65m
notification-controller-58ffd586f7-c4pff   1/1     Running   0          65m
source-controller-6ff87cb475-rhk4g         1/1     Running   0          65m
``` 
9. Для зручності встановимо [CLI клієнт Flux](https://fluxcd.io/flux/installation/)
```sh
✗ curl -s https://fluxcd.io/install.sh | bash
✗ flux get all
```

10. Додамо в репозиторій каталог `demo` та файл `ns.yaml` що містить маніфест довільного `namespace`  
```sh
$ k ai "маніфест ns demo"
✨ Attempting to apply the following manifest:

apiVersion: v1
kind: Namespace
metadata:
  name: demo
```
- Після зміни стану репозиторію контролер Flux їх виявить:
    - зробить git clone  
    - підготує артефакт   
    - виконає узгодження поточного стану IC   

У даному випадку буде створено `ns demo`:
```sh

✗ ydibka@cloudshell:~/flux-gitops/clusters/demo (terraformtest-472215)$ flux logs -f
2025-09-16T16:56:27.979Z info GitRepository/flux-system.flux-system - stored artifact for commit 'Add Flux sync manifests' 
✗ ydibka@cloudshell:~/flux-gitops/clusters/demo (terraformtest-472215)$ k get ns
NAME                 STATUS   AGE
default              Active   67m
demo                 Active   51m
flux-system          Active   67m
kube-node-lease      Active   67m
kube-public          Active   67m
kube-system          Active   67m
local-path-storage   Active   67m
```
Це був приклад як Flux може керувати конфігурацією ІС Kubernetes

11. Застосуємо CLI Flux для генерації маніфестів необхідних ресурсів:
```sh
$ git clone https://github.com/barabidjan/flux-gitops.git
$ cd ../flux-gitops 
$ flux create source git kbot \
    --url=https://github.com/barabidjan/kbot \
    --branch=main \
    --namespace=demo \
    --export > clusters/demo/kbot-gr.yaml
$ flux create helmrelease kbot \
    --namespace=demo \
    --source=GitRepository/kbot \
    --chart="./helm" \
    --interval=1m \
    --export > clusters/demo/kbot-hr.yaml
$ git add .
$ git commit -m"add manifest"
$ git push

$ flux logs -f
2023-12-19T08:58:45.061Z info GitRepository/flux-system.flux-system - stored artifact for commit 'add manifest' 
2023-12-19T08:58:45.466Z info Kustomization/flux-system.flux-system - server-side apply for cluster definitions completed 
2023-12-19T08:58:45.559Z info Kustomization/flux-system.flux-system - server-side apply completed 
2023-12-19T08:58:45.596Z info Kustomization/flux-system.flux-system - Reconciliation finished in 498.659581ms, next run in 10m0s 
2023-12-19T08:59:46.501Z info GitRepository/flux-system.flux-system - garbage collected 1 artifacts 
```

11. Перевіримо наявність пода з нашим PET-проектом та розберемо кластер:
```sh
$ ydibka@cloudshell:~/flux-gitops/clusters/demo (terraformtest-472215)$ k get po -n demo
NAME                         READY   STATUS         RESTARTS   AGE
kbot-helm-7c84f6c995-k4bht   0/1     ErrImagePull   0          21m
ydibka@cloudshell:~/flux-gitops/clusters/demo (terraformtest-472215)$ k describe po -n demo | grep Warning
  Warning  Failed     19m (x5 over 22m)     kubelet            Failed to pull image "barabidjan/helm:v1.1.0-cc72606-amd64": failed to pull and unpack image "docker.io/barabidjan/helm:v1.1.0-cc72606-amd64": failed to resolve reference "docker.io/barabidjan/helm:v1.1.0-cc72606-amd64": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
$ tf state list 
```
