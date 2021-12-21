+++
title = "Github Action自動化部署到kubernetes"
date = "2021-12-21"
author = "Zi-Shane"
tags = ["kubernetes", "CI/CD", "Github Action"]
description = "使用Github Action自動化部署到Azure kubernetes Cluster"
+++

目標
---

我的程式架構如下圖所示，Clent可以通過api來讀寫資料庫中的資料，本文章的目標是將backend部署到kubernetes上的過程交由Github Action自動化完成。  

![](https://i.imgur.com/lNeXLJZ.png)


一般來說database屬於Stateful的Application不太需要重啟，所以database的部分我採用手動的方式來管理（database不會隨著backend程式的重新部署而重啟），[MariaDB的部署請參考這篇的步驟](https://hackmd.io/@7X5kamZwR8yXzSdJtG50Fg/HJ-afK49F)。因此這篇文章主要是討論backend的部分要如何在release新版本時，觸發Github Action自動化部署到新版本到Azure Kubernetes Cluster上。

想要嘗試看看本文的說明的話，程式碼在[這邊](https://github.com/Zi-Shane/api-project/tree/v1.0.1)有興趣可以跟著操作看看。

---

前置準備
---

- Azure環境
    - 架設Container Registry、架設kubernetes參考[官方教學文件(1-3步驟)](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr?tabs=azure-cli)完成即可
- Github Action
    - 建立一個Github Repository

---

架構圖與流程
---

大致流程如下，設定使用者在github增加了新的release時，Github Action會將release的程式碼進行build image、上傳image到Registry、最後將app部署到kubernetes

簡單來說就是把原本手動deploy到k8s上的這個過程給自動化，可以大大減少手動操作的麻煩

![](https://i.imgur.com/lwxkZiv.png)

---

手動部署
---

再開始自動化之前，先來看看手動部署會需要做哪些步驟：

### step1. Build & Push the container image

第一步要將程式包成image並將其上傳到Registry，因此需要先寫好Dockerfile。而中間的`docker login`是只有在第一次登入Registry的時候才需要的。

```
docker build -t tgbotreg.azurecr.io/api-backend:v1.0 .
docker login tgbotreg.azurecr.io
docker push tgbotreg.azurecr.io/api-backend:v1.0
```

這裡附上我的Dockerfile，裡頭有使用到multi-stage build的方式，前半部是做compile的動作，而後半部是把編譯好的binary給包進image，這樣做可以只保留編譯完成的binary來包成image，可以減少image的大小。

{{< code language="Dockerfile" title="" isCollapsed="false">}}
# Compile code
FROM docker.io/library/golang:alpine as builder
WORKDIR /app

COPY . .
RUN go mod download &&\
    go build -o app

# Build image by compiled binary
FROM docker.io/library/alpine
WORKDIR /app
COPY --from=builder /app/app .

EXPOSE 3000

ENTRYPOINT ["./app"]
{{< /code >}}


### step2. Deploy container to the kubernetes cluster

有了image之後就可以丟到k8s上執行了，我使用kustomize來管理yaml，因此準備以下兩的檔案，將其放在同個資料夾中。
- `api-yamls/api-deploy.yaml`: 包含deployment、service
- `api-yamls/kustomization.yaml`: namespace和自動生成secret(database的連線資訊)

接著可以下這個指令`kubectl kustomize ./`就可以看到自動產生的secret和namesapce已經被加入到原本的deployment、service之中了。

kustomize會自動根據`kustomization.yaml`自動生成secret和插入namespace到`api-deploy.yaml`之中。自動生成secret的好處是不用自己轉base64以及避免機密資訊上傳到github上。而另一個好處是透過這種方式可以很方便地將同個服務的資源給綁在一起，不用說deployment、service每個的namespace都要一個個改，比不容易出錯。

kustomize內建在kubectl中，直接使用以下指令來deploy
```bash
# deploy
kubectl apply -k ./

# delete
kubectl delete -k ./
```

{{< code language="yaml" title="api-yamls/api-deploy.yaml" isCollapsed="false">}}
apiVersion: v1
kind: Service
metadata:
  name: api-backend  # Service name
spec:
  type: LoadBalancer  # 使用cloud的loadbalancer分配ip
  ports:
  - port: 3000  # Service開放的port (預設target port也會被設為相同)
  selector:
    app: api-backend  # 此Service作用於哪些pod的
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend-deploy  # Deployment name
spec:
  replicas: 1  # 此Deployment要維持幾個pod
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": linux
      containers:
      - name: api-backend
        image: tgbotreg.azurecr.io/api-backend:v1.0
        env:
        - name: DB_CONN  # ENV in container
          valueFrom:
            secretKeyRef:
              name: mariadb-conn  # 哪個secret
              key: dbconn  # secret中的哪個key
        ports:
        - containerPort: 3000
          name: api-port
{{< /code >}}


{{< code language="yaml" title="api-yamls/kustomization.yaml" isCollapsed="false">}}
secretGenerator:
- name: mariadb-conn
  literals:
  - dbconn=root:yourpassword@tcp(api-mariadb:3306)/nation

resources:
  - api-deploy.yaml
namespace: api-backend  # 加入namespace到resource指定的yaml中
{{< /code >}}


以上就是每次deploy時要做的動作，接下來就要將這些全部改為自動化～

---

Github Action自動化
---

在前面手動部署的步驟中我們是使用kubstomize來管理yaml，好處是可以把機密資訊分開保存(secret跟deployment)，如果要自動化的話一樣需要有secret程式才能正確執行，但我們不想把secret的內容公開到github上，因此secret的部分還是以手動的方式部署到k8s中。

### step1: Deploy DB connection password as secret to k8s

使用`kubectl kustomize ./`可以看到kustomize生成的yaml，把其中secret的部分複製下來存到訂一個檔案`secret.yaml`，記得不要公開到github上！！

{{< code language="yaml" title="manifests/secret.yaml" isCollapsed="false">}}
apiVersion: v1
data:
  dbconn: cm9vdDphYmJjY2NkZGRkQHRjcChhcGktbWFyaWFkYjozMzA2KS9uYXRpb24=
kind: Secret
metadata:
  name: mariadb-conn
  namespace: api-backend
type: Opaque
{{< /code >}}


最後執行以下指令，將secret先部署到k8s中
```
kubectl apply -f manifests/secret.yaml
```

---

### step2: 設定Github Action的workflow

直接看下方的workflow，可以看到`jobs.build.steps`，做了以下幾件事：
- Connect to Azure Container Registry (ACR)
- Container image build and push to a Azure Container Registry (ACR)
- Set the target Azure Kubernetes Service (AKS) cluster
- Create namespace
- Replace image name and tag in deployment.yml
- Deploy app to AKS

其實就是把前面手動的步驟自動化，自動化可以想成有人開了一台VM幫你`git clone`你的repo程式碼並且build它。
Github Action會自動檢查`.github/workflow/xxx.yaml`底下的workflow，只要push到你的repo上，當滿足觸發workflow的條件時，他就會自動執行。

- `on`: 觸發workflow的條件
- `env`: 環境變數，可以使用帶入變數的方式方便撰寫和管理workflow
- `jobs`: 就是workflow實際上要做的事情

`jobs.build.steps.use`是使用[marketplace](https://github.com/marketplace)上別人寫好的模組功能。
而`jobs.build.steps.run`則是直接執行指令。

因此過程中同樣會build image，所以需要Dockerfile、deployment用的yaml。這些檔案分別放在以下路徑
```
./Dockerfile
manifests/deployment.yaml
manifests/service.yaml
.github/workflow/deploy-to-azure.yaml
```

`manifests/deployment.yaml`和`manifests/service.yaml`是使用`kubectl kustomize ./`的結果，其中的imge改成`tgbotreg.azurecr.io/api-backend:LATEST_TAG`，因為每次tag都會不同，所以workflow過程中會把`LATEST_TAG`替換成新的tag。

{{< code language="yaml" title=".github/workflows/deploy-to-azure.yaml" isCollapsed="false">}}
```yaml
# Event that trigger this workflow
on: 
  release:
    types: [published, edited]
    branches:
      - main

# Environment variables available to all jobs and steps in this workflow
env:
  REGISTRY_NAME: tgbotReg
  CLUSTER_NAME: tgbot
  CLUSTER_RESOURCE_GROUP: Telegram-Bot
  NAMESPACE: api-backend
  APP_NAME: api-backend
  
jobs:
  build:  # jobs name (defined by your self)
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@main
    
    # Connect to Azure Container Registry (ACR)
    - uses: azure/docker-login@v1
      with:
        login-server: ${{ env.REGISTRY_NAME }}.azurecr.io
        username: ${{ secrets.REGISTRY_USERNAME }} 
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    # Container build and push to a Azure Container Registry (ACR)
    - run: |
        docker build . -t ${{ env.REGISTRY_NAME }}.azurecr.io/${{ env.APP_NAME }}:${{ github.sha }}
        docker push ${{ env.REGISTRY_NAME }}.azurecr.io/${{ env.APP_NAME }}:${{ github.sha }}
    
    # Set the target Azure Kubernetes Service (AKS) cluster. 
    - uses: azure/aks-set-context@v1
      with:
        creds: '${{ secrets.AZURE_CREDENTIALS }}'
        cluster-name: ${{ env.CLUSTER_NAME }}
        resource-group: ${{ env.CLUSTER_RESOURCE_GROUP }}
    
    # Create namespace if doesn't exist
    - run: |
        kubectl create namespace ${{ env.NAMESPACE }} --dry-run=client -o json | kubectl apply -f -
    
    # Replace image tag in deployment.yml
    - run: |
        sed -i "s/LATEST_TAG/${{ github.sha }}/" manifests/deployment.yml
    
    # Deploy app to AKS
    - uses: azure/k8s-deploy@v1
      with:
        manifests: |
          manifests/deployment.yml
          manifests/service.yml
        images: |
          ${{ env.REGISTRY_NAME }}.azurecr.io/${{ env.APP_NAME }}:${{ github.sha }}
        # imagepullsecrets: |
        #   ${{ env.SECRET }}
        namespace: ${{ env.NAMESPACE }}
{{< /code >}}


這裡還是附上`manifests/deployment.yaml`和`manifests/service.yaml`

{{< code language="yaml" title="manifests/deployment.yaml" isCollapsed="true">}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend-deploy
  namespace: api-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
      - env:
        - name: DB_CONN
          valueFrom:
            secretKeyRef:
              key: dbconn
              name: mariadb-conn
        image: tgbotreg.azurecr.io/api-backend:LATEST_TAG
        name: api-backend
        ports:
        - containerPort: 3000
          name: api-port
      nodeSelector:
        beta.kubernetes.io/os: linux
{{< /code >}}

{{< code language="yaml" title="manifests/service.yaml" isCollapsed="true">}}
apiVersion: v1
kind: Service
metadata:
  name: api-backend
  namespace: api-backend
spec:
  ports:
  - port: 3000
  selector:
    app: api-backend
  type: LoadBalancer
{{< /code >}}


---


參考資料
---
- https://docs.microsoft.com/en-us/azure/aks/kubernetes-action
- https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
- https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr?tabs=azure-cli
