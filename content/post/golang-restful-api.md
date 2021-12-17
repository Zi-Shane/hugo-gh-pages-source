+++
title = "使用Golang來寫一個RESTful API"
date = "2021-12-14"
author = "Zi-Shane"
tags = ["Golang"]
description = "使用Golang寫RESTful API套件使用紀錄"
+++

我的程式架構如下圖所示，Client可以通過api來讀寫資料庫中的資料。這篇文章主要是討論backend的部分要如何實現。

![](https://i.imgur.com/lNeXLJZ.png)

---

工具
---

- Golang
    - [gorilla/mux](https://github.com/gorilla/mux): HTTP router
    - [gorm](https://github.com/go-gorm/gorm): ORM library for Golang
    - [gin](https://github.com/gin-gonic/gin): HTTP web framework
- MariaDB
    - Database的部分採用mariadb，建議使用docker-compose來架設，[這裡有寫好的docker-compose設定（mariadb-phpmyadmin）](https://github.com/Zi-Shane/docker-compose-files)，phpmyadmin也這串好了，可以依自己的需求進行修改（image, username, password...），移動到`mariadb-phpmyadmin`資料夾下`docker-compose up -d`就完成mariadb的架設了

---

net/http
---

golang 內建的套件，網路上可以查到蠻多相關資料的，在使用過程中發現設計api的route有些不方便，例如：

我想要設計一個route，可以指定id來`GET`該筆資料，`net/http`沒辦法做到把url中的參數當成變數的功能，因此api可能要設計成這樣

`api/language/1`  
`api/language/2`  
`api/language/3`  
....  
`api/language/10`

因此查到了`gorilla/mux`這個套件，可以解決我的問題，透過下面這樣的方式，把url中的id這個位置當作input來處理

`api/language/{id}`

---

net/http + gorilla/mux + gorm
---

#### 安裝套件

```
go get -u github.com/gorilla/mux
go get -u gorm.io/gorm
```

#### 程式碼架構

程式碼架構參考[這篇文章](https://medium.com/skyler-record/go-實作-restful-api-2a32210adeaf)，介紹的蠻詳細的。
而本文主要是多了資料庫連線的部分，我處理的方式是把連線資訊獨立出來(`database`資料夾)，並使用`gorm`讀資料庫(`services`資料夾)。而完成的程式碼可以[到這下載](https://github.com/Zi-Shane/api-project/tree/dev-mux)，設定好環境變數和databsase(將`nation.zip`自phpmyadmin匯入資料庫)就可以正常執行。

- `controllers` 處理邏輯
- `routes` 處理API路由
- `services` 讀資料庫
- `database` 與資料庫連線

```
.
|-- routes
|   |-- api-routes.go
|   `-- route-utils.go
|-- controllers
|   |-- api-controller.go
|   `-- controller-utils.go
|-- database
|   `-- database.go
|-- services
|   `-- ReadLanguage.go
|-- go.mod
|-- go.sum
`-- main.go
```

#### 1. 設定路由 `GET api/language/:id`

`main.go`為專案起點，呼叫`routes/route-utils.go`的`routes.NewRouter()`以建立`routes/api-routes.go`中定義的API的路由

{{< code language="go" title="main.go" >}}
package main

import (
	routes "api/routes"
	"net/http"
)

func main() {
	router := routes.NewRouter()         // create a mux Router
	http.ListenAndServe(":3000", router) // start server
}
{{< /code >}}

{{< code language="go" title="routes/route-utils.go" >}}
package routes

import (
	"fmt"
	"net/http"

	"github.com/gorilla/mux"
)

type Route struct {
	Method     string
	Pattern    string
	Handler    http.HandlerFunc
	Middleware mux.MiddlewareFunc
}

var routes []Route

func register(method, pattern string, handler http.HandlerFunc, middleware mux.MiddlewareFunc) {
	routes = append(routes, Route{method, pattern, handler, middleware})
	fmt.Println("Route: ", pattern)
}

// Bind HandlerFunc to Routes
func NewRouter() *mux.Router {
	r := mux.NewRouter()
	for _, route := range routes {
		r.Methods(route.Method).
			Path(route.Pattern).
			Handler(route.Handler)
		if route.Middleware != nil {
			r.Use(route.Middleware)
		}
	}
	return r
}
{{< /code >}}

設定當request打到`GET /api/language/{id}`時，要呼叫的HandlerFunc來處理request。
init()是在package中最先執行的部分，將綁定的部分放在`init()`以確保在`main.go`中call`NewRouter()`時，HandlerFunc和Routes先會綁定在一起才被加到Router之上。

{{< code language="go" title="routes/api-routes.go" >}}
package routes

import (
	"api/controllers"
)

// Configure of API Route
func init() {
	register("GET", "/api/language/{id}", controllers.GetLanguage, nil)
}
{{< /code >}}

---

設計 Route 對應的 Handler Function
---

HandlerFunc主要負責處理Request，若是需要讀取資料庫就會呼叫`services`中的function，並回傳Response

{{< code language="go" title="controllers/api-controller.go" >}}
package controllers

import (
	"api/database"
	"api/services"
	"net/http"

	"github.com/gorilla/mux"
)

// HandlerFunc of `GET /api/language/{id}`
func GetLanguage(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)   // 獲取url參數
	queryId := vars["id"] // 獲取{id}

	// Call function from `services` to get data
	var languages []database.Languages = services.ReadLanguage(queryId)

	// Response data to Client
	response := ApiResponse{"200", languages}
	ResponseWithJson(w, http.StatusOK, response)
}
{{< /code >}}

而Response的內容為一個json的格式，透過自訂義的`ResponseWithJson()`來將struct轉換成json

{{< code language="go" title="ccontrollers/controller-utils.go" >}}
package controllers

import (
	"encoding/json"
	"net/http"
)

type ApiResponse struct {
	ResultCode    string
	ResultMessage interface{}
}

func ResponseWithJson(w http.ResponseWriter, code int, payload interface{}) {
	response, _ := json.Marshal(payload)
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(code)
	w.Write(response)
}
{{< /code >}}

---

讀資料庫
---

`database/database.go`用來建立與databse的連線，連線資訊被放在環境變數中，格式為`user:password@tcp(ip:port)/database-name`。  

{{< code language="go" title="database/database.go" >}}
package database

import (
	"os"
	"time"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

type Languages struct {
	Language_id int
	Language    string
}

var DB *gorm.DB

func init() {
	// dsn format: "user:password@tcp(ip:port)/db"
	// example: "user:password@tcp(127.0.0.1:3306)/nation"
	dsn := os.Getenv("DB_CONN")
	var err error
	DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}

	sqlDB, _ := DB.DB()

	// SetMaxIdleConns sets the maximum number of connections in the idle connection pool.
	sqlDB.SetMaxIdleConns(10)

	// SetMaxOpenConns sets the maximum number of open connections to the database.
	sqlDB.SetMaxOpenConns(100)

	// SetConnMaxLifetime sets the maximum amount of time a connection may be reused.
	//sqlDB.SetConnMaxLifetime(time.Hour)

	DB.AutoMigrate(&Languages{})
}
{{< /code >}}

`services/ReadLanguage.go`的`ReadLanguage()`使用grom執行SQL的查詢指令。

至於connection關閉的問題，github上有對於這個問題的討論，所以本文才會設計將資料庫連線的部分獨立出來，每次處理Request時重複使用已經建立的連線。  

- GORM close() [issue](https://github.com/go-gorm/gorm/issues/3145)

{{< code language="go" title="services/ReadLanguage.go" >}}
package services

import "api/database"

func ReadLanguage(id string) []database.Languages {
	var languages []database.Languages
	// SELECT * FROM language WHERE id=1
	database.DB.Find(&languages, id)

	return languages
}
{{< /code >}}

---

執行程式
---

#### 執行main.go
```shell
go run main.go
```

#### 或是先build成binary再執行
```shell
go build -o app
./app
```

---

`Gin`和`gorilla/mux`差異
---

這裡找到一篇[比較各種golang常見的Web Framework](https://yushuanhsieh.github.io/post/2020-01-21-golang-router/)，主要差別應該在於路由的比對上兩者使用的資料結構不同，Gin在速度上會更快一些。
此外`gorilla/mux`僅是HTTP Router，而`Gin`為Web Framework提供了更多開發後端應用會使用到的功能。

---

用Gin取代gorilla/mux
---

這裡開始使用`Gin`來改寫剛剛的程式，程式碼的架構相同，只需要把`gorilla/mux`和`net/http`的部分改成`Gin`提供的功能取代即可，會改動到的有以下幾處：
- main()
- routes/*
- controllers/*

我是依照github上的範例改寫的，也可以自己先試看看再參考我的程式碼～  
完整程式碼在[這邊](https://github.com/Zi-Shane/api-project/tree/dev)

---

修改main.go
---

`main()`中，改成`Gin`的route和啟動Server的函數，取代前面使用`mux`和`net/http`的部分

<!-- hint:
穿插一點，執行程式的時候會看到第四項的Warning，這個是為了安全的問題，可以依自己的需求加上`router.TrustedPlatform = gin.PlatformCloudflare`等設定，就沒問題了喔 -->

{{< code language="go" title="main.go" >}}
package main

import (
	"api/routes"
)

func main() {
	router := routes.NewRouter()
	// router.TrustedPlatform = gin.PlatformCloudflare
	router.Run(":3000") // listen and serve on 0.0.0.0:8080
}
{{< /code >}}

---

修改Routes
---

`Route`這個struct中，原本使用`net/http`的`HandlerFunc`，這裡改用`gin.HandlerFunc`

`NewRoute()`的部分，把原本回傳的參數(`*mux.Router`)改成的`*gin.Engine`

hint: Middleware的部分暫時略過，因為這個程式很簡單不需要使用到它


{{< code language="go" title="routes/route-utils.go" >}}
package routes

import (
	"github.com/gin-gonic/gin"
)

type Route struct {
	Method  string
	Pattern string
	Handler gin.HandlerFunc
}

var routes []Route

func register(method, pattern string, handler gin.HandlerFunc) {
	routes = append(routes, Route{method, pattern, handler})
}

// Bind HandlerFunc to Routes
func NewRouter() *gin.Engine {
	r := gin.Default()
	for _, route := range routes {
		r.Handle(route.Method, route.Pattern, route.Handler)
	}

	return r
}
{{< /code >}}


這裡也先把Middleware的部分刪掉

{{< code language="go" title="routes/api-routes.go" >}}
package routes

import (
	"api/controllers"
)

// Configure of API Route
func init() {
	register("GET", "/api/language/:id", controllers.GetLanguage)
}
{{< /code >}}

---

修改Controllers
---

因為要把GetLanguage變成`gin.HandlerFunc`所以取參數(id)也改成gin提供的方法。

而回傳給client的json，Gin也有內建的函數(`JSON()`)可以直接使用取代前面自己定義的`ResponseWithJson()`

{{< code language="go" title="controllers/api-controller.go" >}}
package controllers

import (
	"api/database"
	"api/services"
	"os"

	"github.com/gin-gonic/gin"
)

// HandlerFunc of `GET /api/language/{id}`
func GetLanguage(c *gin.Context) {
	queryId := c.Param("id") // 獲取url參數

	// Call function from `services` to get data
	var languages []database.Languages = services.ReadLanguage(queryId)

	// Response data to Client
	c.JSON(200, gin.H{
		"result": languages,
	})
}
{{< /code >}}

---

執行程式
---

我是在VSCode中執行，成功的話會看到像這樣的畫面
![](https://i.imgur.com/unfTUzT.png)


最後提一下，當執行程式的時候會看到第一行的Warning，這個其實是debug mode的關係，其實不用理他沒問題喔！

至於要怎麼把go.mod裡面的`gorilla/mux`刪掉？
只要執行`go mod tidy`就會自動把沒用到的dependencies安全的清掉了喔！
```
go mod tidy
```

---

到這裡大致介紹了`gorilla/mux`、`Gin`、`grom`的使用和差異。下篇將介紹部署到kubernetes上。

參考資料:
---
- [https://medium.com/skyler-record/go-實作-restful-api-2a32210adeaf](https://medium.com/skyler-record/go-實作-restful-api-2a32210adeaf)
- [https://yushuanhsieh.github.io/post/2020-01-21-golang-router/](https://yushuanhsieh.github.io/post/2020-01-21-golang-router/)
- [https://go.dev/blog/using-go-modules#TOC_7.](https://go.dev/blog/using-go-modules#TOC_7.)
