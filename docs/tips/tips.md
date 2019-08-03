## 一些小技巧

### git LF will be replaced by CRLF    

```bash
git config --global core.autocrlf false
```

### git https 每次都需要输入用户名密码的问题

```bash
git config --global credential.helper store\
```

### git从远程拉取所有信息

```bash
git fetch origin --prune
```

### ssh
 
若执行ssh-add /path/to/xxx.pem是出现这个错误:Could not open a connection to your authentication agent，则先执行如下命令即可：
```bash
ssh-agent bash
```

You might need to start ssh-agent before you run the ssh-add command:
```bash
eval `ssh-agent -s`
```

### 虚拟机找不到/mnt/hgfs挂载目录

检测:
```bash
vmware-hgfsclient
```

安装:
```bash
apt-get install open-vm-dkms
apt-get install open-vm-tools-dkms
```

挂载:
```bash
mkdir /mnt/hgfs
chmod /mnt/hgfs

mount -t vmhgfs .host:/ /mnt/hgfs
vmhgfs-fuse .host:/ /mnt/hgfs
```

### 安装zsh

```bash
sudo apt-get update
sudo apt-get install zsh
chsh -s /bin/zsh
sudo vim /etc/passwd/
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

解决windows和linux文件结尾不一致
```bash
cd /Users/<your user>/.oh-my-zsh
find . -name "*.sh" | xargs dos2unix -f
find . -name "*.zsh" | xargs dos2unix -f
dos2unix -f themes/robbyrussel.zsh-theme
cd
dos2unix -f .zshrc
source ~/.zshrc
```
### pip超时

```bash
pip2 install maplotlib==2.2.3 --default-timeout=500
```

### 在python中生成GMT格式

```python
from email.utils import formatdate
 
dt = formatdate(None, usegmt=True)
print(dt)
```

### markdownPad2在win10上HTML渲染组件错误
windows 10系统 需要下载 一个 awesomium_v1.6.6_sdk_win 这是一个 HTML UI ENGINE

下载地址 :
[http://markdownpad.com/download/awesomium_v1.6.6_sdk_win.exe](http://markdownpad.com/download/awesomium_v1.6.6_sdk_win.exe)

### 更改ubuntu源

备份：
```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

检查版本：
```bash
lsb_release -c
```

编辑：
```bash
sudo vim /etc/apt/sources.list
```

添加：
```bash
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse

deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
```

### WSL外观

[ColorTool](https://github.com/microsoft/terminal/releases/tag/1708.14008)

解压后：
```
colortool.exe -b deuteranopia
colortool.exe -b OneHalfDark
colortool.exe -b OneHalfLight
colortool.exe -b solarized_dark
colortool.exe -b solarized_light
```

### pip

下载：
```
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py
```
升级：
```
pip install -U pip
```
使用：
```
python -m pip <pip arguments>
```
配置，linux，`$HOME/.config/pip/pip.conf `,`/etc/pip.conf.`:
```
[global]
timeout = 60
index-url = https://download.zope.org/ppix
```
MAC:`/Library/Application Support/pip/pip.conf`

WIN:`C:\Documents and Settings\All Users\Application Data\pip\pip.ini`,`%HOME%\pip\pip.ini`

On Unix and macOS the configuration file is: `$HOME/.pip/pip.conf`

On Windows the configuration file is: `%HOME%\pip\pip.ini`

### YCM

golang x tools
```
cd ~/.vim/bundle/YouCompleteMe/third_party/ycmd/third_party/go/src/golang.org/x

git clone https://github.com/golang/tools.git
```
 YouCompleteMe/third_party/ycmd/third_party/cregex" does not appear to contain CMakeLists.txt.

```
 rm -rf YouCompleteMe/third_party/ycmd/third_party/cregex 
 git submodule update --init --recursive  at the dir of "YouCompleteMe/third_party/ycmd/third_party"
```