+++
title = "在GitHub上建立一個自己的部落格"
date = "2021-08-14"
author = "Zi-Shane"
tags = ["gh-pages"]
description = " 使用Hugo和GitHub page建立字個的部落格"
+++

- 安裝Hugo
    - Hugo是什麼
    - 安裝
- 建立Blog
- 新增文章
- 增加Disqus留言板和Google Analytics
    - Disqus
    - Google Analytics
- 上傳到GitHub
- 後續
- 參考資料

---

## 安裝Hugo

### Hugo是什麼

Hugo是一個產生靜態網頁的工具，靜態網頁就可以直接部署到Gihub上面建立一個屬於自己的網站

![](https://i.imgur.com/CnT8yfx.png)

### 安裝

mac直接用homebrew安裝
```
brew install hugo
```

Ubuntu安裝的方式

1. 用snap安裝
```
snap install hugo
```

2. 也可以到GitHub上下載[deb安裝檔](https://github.com/gohugoio/hugo/releases/)來安裝
```
curl -OL https://github.com/gohugoio/hugo/releases/download/v0.87.0/hugo_0.87.0_Linux-64bit.deb
sudo dpkg -i hugo_0.87.0_Linux-64bit.deb
```

## 建立Blog

先建立一個空的網站

```
hugo new site <blog-name>
```

當前目錄底下會產生一個`<blog-name>`名稱的資料夾

```
blog
|-- archetypes
|   `-- default.md
|-- config.toml
|-- content
|-- data
|-- layouts
|-- static
`-- themes
```

接著到[官方的主題展示區](https://themes.gohugo.io/)，下載自己喜歡的主題，複製到`themes`底下

```
git clone https://github.com/panr/hugo-theme-hello-friend.git ./themes/hello-friend
```

接著需要設定根目錄的`config.toml`一般來說可以在下載資料夾內可以找到`config.toml`，把內容複製貼上即可

:::info
要注意`config.toml`的`theme`名稱要和剛剛下載到的資料夾名稱相同
:::

```toml
baseurl = "/"
languageCode = "en-us"
theme = "hello-friend"
paginate = 5
```

設定好就可以跑Hugo內建的Server來看結果

```
hugo server
```

這樣就可以看到安裝的主題了，但還因為還沒有新增文章所以空空的

![](https://i.imgur.com/l17ZR66.png)

---

## 新增文章

有兩種方式可以新增文章

1. 用Hugo的指令產生新的markdown file

```
hugo new post/my-test-post.md
```

:::warning
記得要改`my-test-post.md`裡面的draft，要設成false才會看得到文章
:::

2. 或是直接複製`exampleSite`資料夾內的文章來改內容

回到網頁上就可以看到文章了！
而剛剛run的Hugo Server會自動更新內容，所以在寫文章的時候不用一直重開Server滿方便的

![](https://i.imgur.com/PEmLJ9x.png)


---

## 增加Disqus留言板和Google Analytics

### Disqus

Hugo有支援disqus留言板的功能，需要在`layout/partials/comments.html`加上以下內容

```html
{{ template "_internal/disqus.html" . }}
```

然後到根目錄的設定檔`config.toml`加上你的disqus shortname

```toml
baseurl = "/"
languageCode = "en-us"
theme = "hello-friend"
paginate = 5
disqusShortname = "xxxxxx"
```

重新啟動Hugo Server，到任一篇文章的最下方，有看到這樣的提示就完成了!


### Google Analytics

同樣到根目錄的設定檔`config.toml`加上追蹤ID

```toml
baseurl = "/"
languageCode = "en-us"
theme = "hello-friend"
paginate = 5
disqusShortname = "xxxxxx"
googleAnalytics = "UA-xxxxxxxx-x"

```

---

## 上傳到GitHub

再來就是使用hugo指令產生網站，可以看到多出來一個的Public資料夾而最後就是要把這個資料夾push到GitHub上面去

```
hugo
```

在GitHub上建一個Repository，名稱要是`<User-Name>.github.io`，然後把剛剛的public資料夾，push到這個Repository，GitHub會自動Host這個Repository所以在網址列輸入以下網址，就可以看到你的網站成功部屬到GitHub Page

```
https://Zi-Shane.github.io
```

---

## 後續

後續可以再加上gihub action自動build到github page，有興趣可以參考這篇[[3]](https://medium.com/@asishrs/automate-your-github-pages-deployment-using-hugo-and-actions-518b959a51f9)

---

## 參考資料

[1] https://gohugo.io/content-management/comments/#readout

[2] https://support.google.com/analytics/answer/1008080?hl=zh-Hant#zippy=%2C%E6%9C%AC%E6%96%87%E5%85%A7%E5%AE%B9

[3] https://medium.com/@asishrs/automate-your-github-pages-deployment-using-hugo-and-actions-518b959a51f9
