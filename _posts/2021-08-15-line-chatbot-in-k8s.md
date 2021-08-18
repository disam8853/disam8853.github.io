---
title: 使用 Kubernetes 打造具有會員註冊、身份驗證的多功能聊天機器人
description: 為了開發越來越龐大的ChatBot系統，我們可以使用 K8S 作為部署的工具，他可以大大降低維護成本、增加服務擴展性
date: 2021-08-15
categories:
- ChatBot
tags:
- Kubernetes
- blog
- ChatBot
- microservice
---

各位在開發比較龐大的 chatbot 系統時是否會遇到一個問題？就是功能越來越多、程式碼越來越難管理、發生問題時很難找到問題點？這次我們要利用 Kubernetes (k8s) 來打造一個穩定、分散式、擴展性高的 chatbot 系統。

> 本篇程式碼都會放在我的 [GitHub repo](https://github.com/disam8853/line-webhook-in-k8s)
> 
> 這此內容主要為我在 [COSCUP 2021](https://coscup.org/2021/zh-TW/session/ED7M9S) 中所分享的內容

# 1. 目標

這次我們打算做出有以下功能的聊天機器人系統：

- 使用 LINE Login 會員註冊
- 使用 LINE Messaging API 處理聊天室身份驗證
- 定期自動推播訊息給會員

# 2. 設計系統/微服務

為了讓系統更簡潔、清楚、好管理，我們會把整套服務切成以下 5 個微服務：

| 微服務名稱          | 工作                     |
| ------------------- | ------------------------ |
| webhook             | 處理 LINE Webhook events |
| register-web-client | 會員註冊網站前端         |
| register-web-server | 會員註冊網站後端         |
| api-users           | 會員使用者 API           |
| push-msg            | 定期推播訊息的 CronJob   |

## 2.1. 設計思路

1. 使用者會先透過 ingress 進入到 k8s cluster 內的會員註冊網站，並且經由後端傳送註冊會員的資料到 `Users API` 寫入資料庫中。

2. 使用者就能透過 LINE App 發送訊息到 LINE，而 LINE 再發送 webhook event 一樣通過 ingress 給我們的 `webhook` 處理，`webhook` 再與 `Users API` 詢問是否該 LINE UID 為會員並做相對應的處理。

3. 有個 CronJob (`push-msg`) 會自動定期推播訊息給會員。

系統架構圖大概會長這樣：

{% include figure image_path="/assets/images/2021-08-15/coscup.png" alt="Flow chart" caption="系統架構圖" %}

# 3. 開發工具

為了能夠在 local 先測試/開發，我們必須安裝/準備一些工具。

## 3.1. K8S

Kubectl + Kustomize

## 3.2. KInD

我選擇使用 [KInD](https://kind.sigs.k8s.io/) (Kubernetes in Docker) 來作為 k8s cluster，這裡會選擇 KInD 的原因主要是因為能夠模擬多個 nodes 的 cluster 環境，另外也能做到相同目的的還有 [k3d](https://k3d.io/)，也可以選擇使用 k3d。

### 3.2.1. docker-tuntap-osx

如果是使用 Mac OS 的人，必須要安裝 [`docker-tuntap-osx`](https://github.com/AlmirKadric-Published/docker-tuntap-osx)，因為在 Mac OS 裡的 docker 中間是依靠一個 VM `Hyperkit` 來作為橋樑，因此如果要在本機連到 container 內的 k8s cluster 的話就需要依照[外部教學](https://www.thehumblelab.com/kind-and-metallb-on-mac/)串接。

## 3.3. Traefik

使用 [Traefik](https://doc.traefik.io/traefik/) 作為 k8s 的 ingress controller，會選擇使用 Traefik 的主要原因是他提供了許多好用、彈性的 middlewares，只要簡單的在 yaml 裡設定，就能夠使用強大的功能。


# 4. 開始開發

## 4.1. coding

為了能夠有 image 讓 k8s 跑，我們必須先將程式碼打包成 image，然後我這裡是直接 push 到 docker hub，如此一來 k8s 就能直接從 docker hub pull image 了。

## 4.2. 定義 deployment.yaml

設定 `deployment.yaml` 可以讓 k8s 自動地去 deploy 我們要的 docker image。以 `api-users` 這個 microservice 來說，deployment 大概會像這樣：

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: api-users
  labels:
    k8s-app: api-users
spec:
  selector:
    matchLabels:
      k8s-app: api-users
  template:
    metadata:
      labels:
        k8s-app: api-users
        name: api-users
    spec:
      containers:
      - image: disam8853/api-users
        name: api-users
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
           name: api-users-config
```

## 4.3. 定義 service.yaml

此步驟是為了能夠創建 microservice。以 `api-users` 來說，service 大概會像這樣：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: api-users
spec:
  selector:
    k8s-app: api-users
  ports:
    - protocol: TCP
      port: 8080
```

## 4.4. 定義 ingressroute.yaml

為了能讓我們的 ingress controller 可以正確地設定 ingress，我們必須要設定 `ingressroute.yaml`。以 `api-users` 來說，`ingressroute.yaml` 大概會像這樣：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: api-users-ingress
spec:
  entryPoints:
    - web
  routes:
  - match: PathPrefix(`/users`) || PathPrefix(`/register`)
    kind: Rule
    services:
    - name: api-users
      port: 8080
```

## 4.5. 定義 CronJob

不要忘了我們還需要一個 cronjob 來幫助我們自動定期推播訊息給會員，因此我們一樣需要將程式碼打包成 image 後 push 到 docker hub，之後再設定 k8s 的 `cronjob.yaml`：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: push-msg
spec:
  schedule: '*/5 * * * *'
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 60
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: push-msg
              image: disam8853/push-msg
              envFrom:
                - configMapRef:
                    name: webhook-config
                - secretRef:
                    name: webhook-secret
```

以這個例子來說，`*/5 * * * *` 代表每 5 分鐘執行一次這個 cronjob，這個就跟 linux 的 `crontab` 一樣，如果不熟語法的人可以在 [crontab.guru](https://crontab.guru/) 中設定自己的行程表。

## 4.6. Apply

都設定好了之後只要在最上層 `kustmize.yaml` 的路徑執行：

```bash
kustomize build . | k apply -f -
```

就會將我們的設定 apply 到 KInD 的 k8s cluster 裡了，接下來 k8s 就會自動去 docker hub 抓我們的 images，並且依照我們的設定去 deploy、設定 ingress。

## 4.7. 設定 LINE Webhook

為了能夠讓本機 有個對外的網址可以設定 LINE Webhook 的 URL，我們可以使用免費的 reserve proxy 來幫助開發，這裡我建議使用 [`localtunnel`](https://github.com/localtunnel/localtunnel)，因為 `localtunnel` 免費版就能夠自己設定 subdomain name，不像 ngrok 需要付費才能有此功能。

# 5. Demo

Demo 影片可以參考我之前在 COSCUP 2021 所分享的影片 (15:15 處)：

{% include video id="VpG7Nhe5nCg?start=915" provider="youtube" %}

# 6. 總結

使用 k8s 來開發較為龐大的系統可以大大的減少維護成本，例如當有問題發生時，可以利用 microservice 的優點，快速的排查問題原因，提升 debug 效率。

現在的 chatbot 基本上都沒有身份驗證的功能，因此需要開發者自己維護一個會員系統，會員系統又包含了註冊、登入等等服務，如此一來如果使用 k8s 來開發、部署，就可以讓他們完美的配合、溝通。不但之後可以再以這套基礎加上其他服務 (ex: 購物車)，更能處理大量請求的尖峰時段。