+++
title = "在VSCode上使用jupyter notebook"
date = "2020-10-01"
author = "Zi-Shane"
tags = ["python", "vscode"]
description = "如何讓VSCode可以像jupyter notebook的方式執行程式碼"
+++

## 安裝 Python 擴充套件

![](https://i.imgur.com/M3Ier29.png)

---

## 安裝 `ipykernel` 套件

```pip
pip install ipykernel
```

![](https://i.imgur.com/xp4obd7.png)

---

## 設定 `jupyterServerURI`

到設定頁面複製設定

![](https://i.imgur.com/q33Xq5j.png)

貼上到workspace的設定檔(`.vscode/setting.json`)中

![](https://i.imgur.com/boNjUSd.png)

:::info
**若是出現下面的錯誤，同樣重新設定jupyterServerURI就可以解決了** 
![](https://i.imgur.com/ION2o6D.png)
:::

---

## 將程式分成多個區塊

用`#%%`來將程式分成多個區塊

![](https://i.imgur.com/8LqtNZu.png)

同時按`shift`+`enter`執行區塊

![](https://i.imgur.com/138WNlt.png)
