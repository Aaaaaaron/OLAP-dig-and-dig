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

### 安装homebrew cask(已废弃, 现在直接装就行)

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
brew install htop
brew install node
brew install --cask shiftit
brew install --cask google-chrome
# 买的 9 的, 现在 brew 会装最新版也就是 10, 同理 Alfread
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

配置快捷键

<img src="./Mac-initalization/image-20221008145858555.png" alt="image-20221008145858555" style="zoom:50%;" />

<img src="./Mac-initalization/image-20221008150107285.png" alt="image-20221008150107285" style="zoom:50%;" />

### 配置定时任务

举例 kinit
1. 配置 kinit 自动输入密码 `kinit --keychain xxx@xxx`
2. 编写脚本, 并且赋予可执行权限 

```bash
# kinit_crontab.sh
/usr/bin/kinit --keychain xxx@xxx
echo "done kinit"
```

3. 编写launchctl脚本

- cd ~/Library/LaunchAgents
- vim com.user.kinit.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <!-- 一定以com 开头 -->
    <string>com.user.kinit</string>
    <key>Program</key>
    <string>/Users/aron/kinit_crontab.sh</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/aron/kinit_crontab.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <!-- 60s -->
    <integer>60</integer>
    <key>StandardOutPath</key>
    <string>/Users/aron/log/kinit_run.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/aron/log/kinit_run.err</string>
</dict>
</plist>

```

4. launchctl定时执行脚本

```sh
cd ~/Library/LaunchAgents
launchctl load -w com.user.kinit.plist
卸载脚本：launchctl unload com.user.kinit.plist
```

### 配置 gdb init

`vim ~/.gdbinit`

```py
python
import sys
sys.path.insert(0, '~/gcc-10/share/gcc-10.2.0/python')
# sys.path.insert(0, '~/gcc-9/share/gcc-9.3.0/python')
from libstdcxx.v6.printers import register_libstdcxx_printers
register_libstdcxx_printers (None)

sys.path.insert(0, '~/clang/gdb')
from libcxx.printers import register_libcxx_printer_loader
register_libcxx_printer_loader()
end
```

