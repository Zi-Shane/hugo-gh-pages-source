+++
title = "使用Golang來寫一個RESTful API - Part2"
date = "2021-12-17"
author = "Zi-Shane"
tags = ["Golang", "RESTful API", "後端"]
description = "用Gin和Gorm來做出一個有CRUD功能的RESTful API"
+++

我的程式架構如下圖所示，Client可以通過api來讀寫資料庫中的資料。這篇文章主要是討論backend的部分要如何實現。本實作會使用Gin和Gorm來完成，Database則使用MariaDB。

這篇文章是延續上[一篇的內容](https://zi-shane.github.io/post/golang-restful-api/)，多完成了對資料庫進行Create, Update, Delete以及Join這種比較複雜的查詢。完成的程式碼可以參考我的[github](https://github.com/Zi-Shane/api-project/tree/v1.0.1)。

#### 架構圖

![](https://i.imgur.com/lNeXLJZ.png)

#### 資料庫Scheme

資料來源：[https://www.mariadbtutorial.com/getting-started/mariadb-sample-database/](https://www.mariadbtutorial.com/getting-started/mariadb-sample-database/)

![](https://i.imgur.com/0C5yAY3.png)


#### 完成的成果有以下功能：

```
GET    /api/GetLanguage/:id               --> 根據id讀出對應的Language 
GET    /api/GetLanguageRange/:start/:end  --> 根據給定id範圍，讀出所有Language 
GET    /api/GetCountryUselanguage/?counry --> 根據query的country，讀出其所使用語言
POST   /api/AddLanguage                   --> 根據POST的Body(json)來新增資料 
DELETE /api/DeleteLanguage/:language      --> 根據language刪除該筆資料 
PUT    /api/UpdataLanguage                --> 根據PUT的Body(json)更新資料
```

---

開發環境
---

- Golang
    - [gorm](https://github.com/go-gorm/gorm): ORM library for Golang
    - [gin](https://github.com/gin-gonic/gin): HTTP web framework
- MariaDB
    - Database的部分採用MariaDB，資料使用的是網路上的[sample database](https://www.mariadbtutorial.com/getting-started/mariadb-sample-database/)可以直接下載。
    建議可以使用docker-compose來架設，[這裡有寫好的docker-compose設定（mariadb-phpmyadmin）](https://github.com/Zi-Shane/docker-compose-files)，phpmyadmin也這串好了。  

        使用方式：  
        可以自己修改image, username, password，最後`cd`到`mariadb-phpmyadmin`資料夾下`docker-compose up -d`就可以啟動一個MariaDB了。
- API測試工具
    - [Thunder Client](https://www.thunderclient.io): 是一個VSCode的Extension，可以用來取代Postman而且還不用額外安裝其它程式

---

程式碼架構
---

程式進入點為main.go，資料庫連線設定在`database`資料夾，而我會將整個API拆成三個部分，分別放在不同的資料夾：
- `routes`: 設定api的路徑，與對應的function
- `controllers`: 處理api收到request（route對應到的function）
- `services`: 對資料庫操作

```
.
|-- routes
|   |-- api-routes.go
|   `-- route-utils.go
|-- controllers
|   |-- language-controller.go
|   `-- hostInfo-controller.go
|-- services
|   `-- Language-service.go
|-- database
|   |-- scheme.go
|   `-- database.go
|-- go.mod
|-- go.sum
`-- main.go
```

---

實作程式
---

將資料庫的四種操作分別對應到HTTP的四種方法。而我所設計的API提供這四種功能來對資料庫進行操作


| 資料庫功能          | HTTP Method |
|:------------------- |:-----------:|
| 查詢Read (SELECT)   |     GET     |
| 新增Create (INSERT) |    POST     |
| 更新Update (UPDATE) |     PUT     |
| 刪除Delete (DELETE) |   DELETE    |

---

開始實作
---

### 查詢Read

查詢可以分為簡單的查詢
{{< code language="sql" title="" >}}
SELECT * FROM `language` WHERE id=1
{{< /code >}}

或是JOIN這種多個Table的複雜查詢
{{< code language="sql" title="" >}}
SELECT C.name AS Country_Name, L.language FROM `countries` AS C
JOIN `country_languages` AS CL
ON CL.country_id = C.country_id
JOIN `languages` AS L
ON CL.language_id = L.language_id
{{< /code >}}

#### 簡單的查詢

我們可以使用Gorm來對資料庫下指令，`database.DB`是我們的程式和databse建立的連線。

以這個查詢為例，這個例子應該蠻好理解的`Where()`是條件，`Find(&languages)`會把查詢結果放進去`languages`變數之中。
- SQL
{{< code language="sql" title="" >}}
SELECT * FROM `language` WHERE `Language_id` BETWEEN 10 AND 20
{{< /code >}}

- Code
{{< code language="go" title="controllers/language-controller.go" >}}
func ReadLanguages(startId string, endId string) ([]database.Languages, error) {
	var languages []database.Languages
    
	// SELECT * FROM `language` WHERE `Language_id` BETWEEN 10 AND 20
	result := database.DB.Where("Language_id BETWEEN ? AND ?", startId, endId).Find(&languages)

	return languages, result.Error
}
{{< /code >}}


#### 複雜查詢

以這個查詢為例，有三個table做JOIN，分別是`countries`、`languages`和`country_languages`，由`Joins()`指定JOIN的條件，而`Where()`是SELECT...FROM...WHERE的條件，最後`Scan(&r)`把結果存入`r`這個變數之中。

提醒一點，structure的欄位要和查詢的結果column名稱相同，也就是要根據`C.name AS CountryName`的部分～

- SQL
{{< code language="sql" title="" >}}
SELECT C.name AS Country_Name, L.language FROM `countries` AS C
JOIN `country_languages` AS CL
ON CL.country_id = C.country_id
JOIN `languages` AS L
ON CL.language_id = L.language_id
{{< /code >}}

- Code
{{< code language="go" title="services/Language-service.go" >}}
func GetCountryUsedLanguages(country string) ([]database.CountryUesdLanguage, error) {
	var r []database.CountryUesdLanguage

	result := database.DB.
		Table("countries AS C").
		Select("C.name AS CountryName, L.language AS Language").
		Joins("JOIN country_languages AS CL ON CL.country_id = C.country_id").
		Joins("JOIN languages AS L ON CL.language_id = L.language_id").
		Where("C.name = ?", country).
		Scan(&r)

	return r, result.Error
}
{{< /code >}}
{{< code language="go" title="database/scheme.go" >}}
type CountryUesdLanguage struct {
	CountryName string
	Language    string
}
{{< /code >}}

---

### 新增Create

`InsertLanguage()`的部分，Gorm會根據structure的名稱去選擇Table


- SQL
{{< code language="sql" title="" >}}
INSERT INTO `Language` (`Language_id`, `Language`) VALUES (500, "Test1"), (500, "Test1")
{{< /code >}}


- code
{{< code language="go" title="services/Language-service.go" >}}
func InsertLanguage(items []database.Languages) (int64, error) {
	// INSERT INTO `Language` (`Language_id`, `Language`) VALUES (500, "Test1"), (500, "Test1")
	result := database.DB.Create(items)

	return result.RowsAffected, result.Error
}
{{< /code >}}

---

### 刪除Delete

`DeleteLanguage()`的部分，`Delete(&database.Languages{})`DELETE並不需要回傳資料，其用意應該跟上面（新增Create）的意思差不多，Gorm會根據structure的名稱去選擇Table，

- SQL
{{< code language="sql" title="" >}}
DELETE FROM `Language` WHERE `Language` = "Test1"
{{< /code >}}

- code
{{< code language="go" title="services/Language-service.go" >}}
func DeleteLanguage(languageName string) (int64, error) {
	// DELETE FROM `Language` WHERE `Language` = "Test1"
	result := database.DB.Where("Language = ?", languageName).Delete(&database.Languages{})

	return result.RowsAffected, result.Error
}
{{< /code >}}

---

### 更新Update

- SQL
{{< code language="sql" title="" >}}
UPDATE `Language` SET `Language` = "UpdatedLanguage" WHERE `Language_id` = 500
{{< /code >}}

- code
{{< code language="go" title="services/Language-service.go" >}}
func UpdateLanguage(item database.Languages) (int64, error) {
	// UPDATE `Language` SET `Language` = "UpdatedLanguage" WHERE `Language_id` = 500
	result := database.DB.Model(&database.Languages{}).Where("Language_id = ?", item.Language_id).Update("Language", item.Language)

	return result.RowsAffected, result.Error
}
{{< /code >}}

---

### 處理Request

#### `GET api/GetLanguageRange/:start/:end`

範例：查詢Language_id為10~20的資料
```
http://localhost:3000/api/GetLanguageRange/10/20
```

當有Request戳到`GET api/GetLanguageRange/:start/:end`時，會執行`GetLanguages()`這個HandleFunc()。

{{< code language="go" title="routes/api-routes.go" >}}
func setupLanguageRoute() {
	register("GET", "/api/GetLanguage/:id", controllers.GetLanguage)
	register("GET", "/api/GetLanguageRange/:start/:end", controllers.GetLanguages)
	register("GET", "/api/GetCountryUselanguage", controllers.GetCountryUesdLanguages)
}
{{< /code >}}

由`c.Param()`來取出參數，帶入用Gorm寫好的function來取得SQL的查詢結果

{{< code language="go" title="controllers/language-controller.go" >}}
func GetLanguages(c *gin.Context) {
	// get URL parameters
	startId := c.Param("start")
	endId := c.Param("end")
	// Call function to get data from database
	languages, err := services.ReadLanguages(startId, endId)
	if err != nil {
		c.JSON(200, gin.H{"message": err.Error()})
	} else {
		// Response result to Client
		c.JSON(200, gin.H{"message": "success", "result": languages})
	}
}
{{< /code >}}

回傳果如下
{{< code language="json" title="" >}}
{
  "message": "success",
  "result": [
    {
      "Language_id": 10,
      "Language": "Ambo"
    },
    {
      "Language_id": 11,
      "Language": "Chokwe"
    },
    {
      "Language_id": 12,
      "Language": "Kongo"
    },
    {
      "Language_id": 13,
      "Language": "Luchazi"
    }
  ]
}
{{< /code >}}

---

#### `GET    /api/GetCountryUselanguage/?counry`

範例：查詢Canada所使用的Language為何？
```
http://localhost:3000/api/GetCountryUselanguage/?country=Canada
```

當有Request戳到`GET api/GetCountryUselanguage/?counry`時，會執行`GetCountryUesdLanguages()`這個HandleFunc()

和前一個差異在於`?country=Canada`這個稱為一個query，可以視為一組key, value，可以使用`c.Query("country")`取出值

{{< code language="go" title="controllers/language-controller.go" >}}
func GetCountryUesdLanguages(c *gin.Context) {
	country := c.Query("country")
	var result []database.CountryUesdLanguage
	result, err := services.GetCountryUsedLanguages(country)
	if err != nil {
		c.JSON(200, gin.H{"message": err.Error()})
	} else {
		// Response data to Client
		c.JSON(200, gin.H{"message": "success", "result": result})
	}
}
{{< /code >}}

回傳果如下
{{< code language="json" title="" >}}
{
  "message": "success",
  "result": [
    {
      "CountryName": "Canada",
      "Language": "Dutch"
    },
    {
      "CountryName": "Canada",
      "Language": "English"
    },
    ...
    {
      "CountryName": "Canada",
      "Language": "Chinese"
    },
    {
      "CountryName": "Canada",
      "Language": "Eskimo Languages"
    },
    {
      "CountryName": "Canada",
      "Language": "Punjabi"
    }
  ]
}
{{< /code >}}

---

#### POST   /api/AddLanguage

範例：新增n筆Language資料
```
http://localhost:3000/api/AddLanguage
```
POST帶的Body如下，為一個json含有多筆要新增的資料
{{< code language="json" title="" >}}
[
    {
        "Language_id": 500,
        "Language": "Test500"
    },
    {
        "Language_id": 501,
        "Language": "Test501"
    }
]
{{< /code >}}


當有Request戳到`POST   /api/AddLanguage`時，會執行`AddLanguage()`這個HandleFunc()

這裡使用`Bind(&m)`將body內的json給parse到structure之中

{{< code language="go" title="controllers/language-controller.go" >}}
func AddLanguage(c *gin.Context) {
	var m []database.Languages
	c.Bind(&m)
	rowsAffected, err := services.InsertLanguage(m)
	if err != nil {
		c.JSON(200, gin.H{"message": err.Error()})
	} else {
		c.JSON(200, gin.H{"message": "success", "rowsAffected": rowsAffected})
	}
}
{{< /code >}}

---

#### DELETE /api/DeleteLanguage/:language

範例：刪除所有Language=Test500的資料
```
http://localhost:3000/api/RemoveLanguage/Test501
```

當有Request戳到`DELETE /api/DeleteLanguage/:language`時，會執行`RemoveLanguage()`這個HandleFunc()

這裡與前面查詢取得URL參數的方法相同，一樣使用`c.Param()`

{{< code language="go" title="controllers/language-controller.go" >}}
func RemoveLanguage(c *gin.Context) {
	queryId := c.Param("language")
	rowsAffected, err := services.DeleteLanguage(queryId)
	if err != nil {
		c.JSON(200, gin.H{"message": err.Error()})
	} else {
		var mesg string
		if rowsAffected == 0 {
			mesg = "Language not found"
		} else {
			mesg = "success"
		}
		c.JSON(200, gin.H{"message": mesg, "rowsAffected": rowsAffected})
	}
}
{{< /code >}}

---

#### PUT    /api/UpdataLanguage

範例：更新Language_id=xxx的資料
```
http://localhost:3000/api/UpdateLanguage
```

POST帶的Body如下，為一個json表示要將Language_id=500的資料更新成"Updated500"
{{< code language="json" title="" >}}
{
    "Language_id": 500,
    "Language": "Updated500"
}
{{< /code >}}



當有Request戳到`PUT    /api/UpdataLanguage`時，會執行`UpdateLanguage()`這個HandleFunc()

這裡與前面新增資料時的方法相同，一樣使用`c.Bind(&m)`把json給parse到structure之中

{{< code language="go" title="controllers/language-controller.go" >}}
func UpdateLanguage(c *gin.Context) {
	var m database.Languages
	c.Bind(&m)
	rowsAffected, err := services.UpdateLanguage(m)
	if err != nil {
		c.JSON(200, gin.H{"message": err.Error()})
	} else {
		c.JSON(200, gin.H{"message": "success", "rowsAffected": rowsAffected})
	}
}
{{< /code >}}


---

以上就是用Gin來開發一個的API的簡單範例～

下一篇文章考慮記錄一下我使用Github Action自動化，將這個API部署到Azure k8s的方式。

敬請期待～～


