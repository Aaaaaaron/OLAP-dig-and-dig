---
title: Mac initalization...
date: 2022-09-17 18:18:16
tags:
---
0. 最重要的几个软件配置:
- Oh-my-zsh/.zshrc
- Sublime Text3
    - Alignment
    - AlignTab
    - Compare Side-By-Side
    - Expand Region
    - Insert Nums
    - Markdown Editing
    - Package Control
    - Package ResourceViewer
    - PackageSync
    - Pretty JSON
    - SQL Beautifier
    - Table Editor
    - TerminalView
    - CodeFormatter
    - SFTP
    - Terminal
    - OmniMarkupPreviewer
- VS-code (有自动配置同步)
- Hexo 博客
- IDEA 全家桶(有自动配置同步)
- Item2
- Vimrc
- Keyboard Maestro
- Alfread
- Typora
- 输入法的配置(搜狗/账号QQ邮箱)
- Mendeley Desktop 管理论文的
- Calibre
- 音乐备份(Foobar2000)

1. 安装Command Line Tools for Xcode
xcode-select --install

2. 安装包管理工具Homebrew
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

brew doctor
brew git
brew item2

3. 安装homebrew cask(已废弃)
```shell
brew tap homebrew/cask
brew install brew-cask

brew cask install google-chrome // 安装 Google 浏览器
brew update && brew upgrade brew-cask && brew cleanup // 更新
```

4. 查看已安装软件
`ls -l /Applications | awk '{print $3"\t", $0}' | sort > ~/Desktop/AppList.txt`

5. 重要的一些软件:
- Tadama(app store)
- Contexts
- eul(机器信息cpu mem 什么的)
- 0 中的那些软件
- PDF Expert
- Sequel Pro(mysql client)
- HandShaker
- CheatSheet(用的不多)
- Fluid(网页 to app)
- Timing(看自己电脑上花了多少时间)/Qbserve(同类)
- Go2Shell
- IINA
- JD-GUI
- The Unarchiver
- kindle
- projector/IDEA
- QQ/wechat/QQ music
- Snipaste
- Spectacle