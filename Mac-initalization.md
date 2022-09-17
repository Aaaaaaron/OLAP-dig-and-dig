---
title: Mac initalization...
date: 2022-09-17 18:18:16
tags:
---
### 最重要的几个软件配置:
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

### 安装Command Line Tools for Xcode
xcode-select --install

### 安装包管理工具 Homebrew
`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
brew doctor

brew install git

brew install iterm2
brew install alfred
brew install appcleaner
brew install cheatsheet
brew install dropbox
brew install google-chrome
brew install onepassword
brew install sublime-text
brew install totalfinder
brew install --cask keyboard-maestro
htop
node
### 安装homebrew cask(已废弃, 现在直接转就行)
```shell
brew tap homebrew/cask
brew install brew-cask

brew cask install google-chrome // 安装 Google 浏览器
brew update && brew upgrade brew-cask && brew cleanup // 更新
```

### 查看已安装软件
`ls -l /Applications | awk '{print $3"\t", $0}' | sort > ~/Desktop/AppList.txt`

### 重要的一些软件:
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

### 脚本
```bash
brew install --cask google-chrome
brew install --cask keyboard-maestro
brew install --cask wechat
# vim ~/Library/Application\ Support/.com.contextsformac.Contexts.plist
brew install --cask contexts
brew install --cask qqmusic
# sougou拼音这个还要安装
brew install --cask sogouinput 
brew install --cask typora
brew install --cask spectacle
brew install --cask qq
brew install --cask sublime-text
brew install --cask visual-studio-code
brew install --cask calibre
brew install --cask foobar2000
brew install --cask mendeley
brew install --cask intellij-idea
brew install --cask clion
brew install --cask iina
brew install --cask snipaste
brew install --cask handshaker
brew install --cask eul
brew install --cask thunder
brew install --cask pdf-expert
brew install --cask sequel-pro
brew install --cask fluid
brew install --cask jd-gui
brew install --cask projector
```