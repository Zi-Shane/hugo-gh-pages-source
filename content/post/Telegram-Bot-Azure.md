+++
title = "Deploy Telegram Bot to Azure"
date = "2021-11-03"
author = "Zi-Shane"
tags = ["Azure", "Golang"]
description = "文中會使用Go來寫一個Telegram Bot，並將其部署到Azure上"
+++

這篇文章會用[telegram-bot-api](https://github.com/go-telegram-bot-api/telegram-bot-api)這個套件在github上的範例程式（是一個echo server），最後把部署到Azure的AKS上去

1. Create your Telegram Bot
2. Build container image
3. Create a Registry on Azure
4. Push container image to Azure Container Registry(ACR)
5. Create a k8s cluster
6. Deploy image to k8s
7. Future work

---

## Create your Telegram Bot

如果到google搜尋的話，第一個看到應該是這個套件[telegram-bot-api](https://github.com/go-telegram-bot-api/telegram-bot-api)，他算是一個把[Telegram Bot API文件中的method](https://core.telegram.org/bots/api#using-a-local-bot-api-server)包好，讓你可以直接call他寫好的function，幫助你快速做出Bot的套件。

仔細去看code會發現，他其實會幫你產生Request去跟Telegram Bot API要資料回來

以getUpdate為例，從程式碼對照官方API文件([getUpdate](https://core.telegram.org/bots/api#getupdates))可以看到，他們是一樣的東西

![照官方API文件(getUpdate)](https://i.imgur.com/7eD5TL5.png)
官方API文件(getUpdate)

![](https://i.imgur.com/euE3mKT.png)
套件中的GetUpdate函數

這篇文章會用github給的[範例程式](https://github.com/go-telegram-bot-api/telegram-bot-api#example)，這是一個echo server，最後要部署到Azure的AKS上去

首先，安裝套件
> First, ensure the library is installed and up to date by running go get -u github.com/go-telegram-bot-api/telegram-bot-api.

改好`TOKEN`之後，可以跑看看這個程式，再到Telegram上發訊息給你的Bot，Bot就會回覆相同的訊息給你

至於`TOKEN`我自己是把它存在一個`.env`檔案中，再把他ignore掉才不會不小心上傳到github上去，需要的話可以直接使用下面這段程式碼，主要使改成從環境變數來讀取`TOKEN`

- `.env`
```
TELEGRAM_APITOKEN=123456:yaaaaaaaaaaa
```

{{< code language="go" title="app.go" >}}
package main

import (
	"log"
	"os"

	tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api"
	"github.com/joho/godotenv"
)

func main() {
	envs_path := "./secret/.env"
	err := godotenv.Load(envs_path)
	if err != nil {
		log.Fatal("Error loading .env file")
	} else {
		log.Printf("Loaded ENV from %s", envs_path)
	}
	bot, err := tgbotapi.NewBotAPI(os.Getenv("TELEGRAM_APITOKEN"))
	if err != nil {
		log.Panic(err)
	}

	bot.Debug = true

	log.Printf("Authorized on account %s", bot.Self.UserName)

	// Create a new UpdateConfig struct with an offset of 0. Offsets are used
	// to make sure Telegram knows we've handled previous values and we don't
	// need them repeated.
	updateConfig := tgbotapi.NewUpdate(0)

	// Tell Telegram we should wait up to 30 seconds on each request for an
	// update. This way we can get information just as quickly as making many
	// frequent requests without having to send nearly as many.
	updateConfig.Timeout = 30

	// Start polling Telegram for updates.
	updates, err := bot.GetUpdatesChan(updateConfig)

	// Let's go through each update that we're getting from Telegram.
	for update := range updates {
		// Telegram can send many types of updates depending on what your Bot
		// is up to. We only want to look at messages for now, so we can
		// discard any other updates.
		if update.Message == nil {
			continue
		}

		// Now that we know we've gotten a new message, we can construct a
		// reply! We'll take the Chat ID and Text from the incoming message
		// and use it to create a new message.
		msg := tgbotapi.NewMessage(update.Message.Chat.ID, update.Message.Text)
		// We'll also say that this message is a reply to the previous message.
		// For any other specifications than Chat ID or Text, you'll need to
		// set fields on the `MessageConfig`.
		msg.ReplyToMessageID = update.Message.MessageID

		// Okay, we're sending our message off! We don't care about the message
		// we just sent, so we'll discard it.
		if _, err := bot.Send(msg); err != nil {
			// Note that panics are a bad way to handle errors. Telegram can
			// have service outages or network errors, you should retry sending
			// messages or more gracefully handle failures.
			panic(err)
		}
	}

}
{{< /code >}}

---

## Build container image

再來要產生container image，使用以下Dockerfile產生Telegram Bot的image，要用來deploy到AKS之上

而我的資料夾架構長得像這樣：
```
Telegram-Bot
├── Dockerfile
├── app.go
├── go.mod
├── go.sum
├── secret
└── tgbot.yaml
```

我是使用podman，如果用docker的話也沒問題，把`podman`改成`docker`即可

```
podman build -t tgbot:v1 .
```

- Dockerfile
```dockerfile
FROM docker.io/library/golang
RUN mkdir -p /app
WORKDIR /app
COPY . .
RUN go mod download &&\
    go build -o app
ENTRYPOINT ["./app"]
```

可以先在local端自己跑起來試看看

```
podman run --name tgbot tgbot:v1
```

理論上會和[第一步的結果](##Create_your_Telegram_Bot)相同

---

## Create a Registry on Azure

要部署到Azure之前的準備工作，本文是參考Azure的[這份教學文件](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr?tabs=azure-cli)來做的，會使用到[Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

這裡所做的事情是在Azure建一個類似Docker Hub的地方，讓你可以上傳你的container images

這部分我是用[Azure Portal](https://portal.azure.com/)用GUI來做，所做的事情相對簡單，主要是做以下兩個。

- 建立Resource Group和選擇Location
- 建立Azure Container Registry

![](https://i.imgur.com/zmrDV64.png)

當然如果想用cli做也可以參考[這份教學文件](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-acr?tabs=azure-cli)

---

## Push container image to Azure Container Registry(ACR)

再來要把Bot的image給push到剛剛建立的Registry上面

到Azure Portal上`Settings -> Access keys`把Admin user的選項打開，就可以透過帳號密碼上傳自己的image

![](https://i.imgur.com/JHe2l7P.png)

回到自己電腦的terminal，用podman登入剛剛得到的username和password，並上傳image

```bash
podman login
# username:
# password:

podman image tag tgbot:v1 tgbotreg.azureacr.io/tgbot:v1

podman push tgbotreg.azureacr.io/tgbot:v1
```

等待上傳完成...（note: 另一篇文章會改成更好的方式來建立image）
到`Service -> Repository`可以看到上傳好的image

![](https://i.imgur.com/Gejp07v.png)


---

## Create a k8s cluster

再來就是使用AKS建置一個k8s的cluster了，一樣參考[這份教學文件](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli)

只是這部分我是使用Azure Cli來完成，因為Azure Portal上遇到一些問題...

直接下指令，Resource Group要和Registry相同，Cluster Name可以隨意取

```azure
az aks create \
    --resource-group Telegram-Bot \
    --name tgbot \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr tgbotReg
```

稍微等待他跑完...

![](https://i.imgur.com/izYiJUU.png)

用以下指令來設定`kubectl`，記得指對Resource Group和Cluster Name

```azure
az aks get-credentials --resource-group Telegram-Bot --name tgbot
```

接著就可以用`kubectl`來操作cluster了～

![](https://i.imgur.com/pH3oLeM.png)

---

## Deploy image to k8s

我是使用[k8s yaml的生成器](https://k8syaml.com)來產生部署用的yaml file(tgbot.yaml)，產生的Deployment，要修改3個地方
- 第4行：取一個Deployment名稱
- 第18行：取一個Container名稱
- 第19行：指定container image

部署方式有兩種:
- 透過kubectl
```
kubectl apply -f tgbot.yaml
```
or
- 用Azure Portal，複製貼上`yaml file`
![](https://i.imgur.com/P2qWeBq.png)


- tgbot.yaml

```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tgbot-deploy
spec:
  selector:
    matchLabels:
      octopusexport: OctopusExport
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        octopusexport: OctopusExport
    spec:
      containers:
        - name: tgbot
          image: 'tgbotreg.azurecr.io/tgbot:v1'
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - web
                topologyKey: kubernetes.io/hostname

```

---

## Future work

這篇文章主要是部署一個Go程式到Azure的k8s cluster上，這中間有許多可以在改善或是延伸的地方，像是使用CI/CD來自動化build image、不使用套件而改用webhook的方式來寫Telegram Bot等等

---

參考文件：
- https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-application
- https://pkg.go.dev/github.com/go-telegram-bot-api/telegram-bot-api@v4.6.4+incompatible#readme-example

