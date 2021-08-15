+++
title = "neovim安裝紀錄"
date = "2021-01-22"
author = "Zi-Shane"
tags = ["neovim"]
description = "neovim安裝及設定"
+++

{{< code language="bash" title="1" >}}
brew install neovim
{{< /code >}}

## 安裝neovim
```
brew install neovim
```

---

## 安裝[vim-plugin](https://github.com/junegunn/vim-plug)
```
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
```

---

## 安裝plugin
`esc`，進入命令模式（Command mode），用`:PlugInstall`安裝Plugin
![](https://i.imgur.com/BsrzwPe.png)

設定檔：[https://github.com/Zi-Shane/init.vim](https://github.com/Zi-Shane/init.vim)

---

## YouCompleteMe報錯
下次打開nvim遇到YouCompleteMe的錯誤，用以下指定解決
```
python3 -m pip install --user --upgrade pynvim

brew install cmake

python3 ~/.config/nvim/plugged/YouCompleteMe/install.py --clang-completer

```
