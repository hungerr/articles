## 一些小技巧

### git LF will be replaced by CRLF    

```
git config --global core.autocrlf false
```

### git https 每次都需要输入用户名密码的问题

```
git config --global credential.helper store\
```

### git从远程拉取所有信息

```
git fetch origin --prune
```

### ssh
 
若执行ssh-add /path/to/xxx.pem是出现这个错误:Could not open a connection to your authentication agent，则先执行如下命令即可：
```
ssh-agent bash
```

You might need to start ssh-agent before you run the ssh-add command:
```
eval `ssh-agent -s`
```

### 虚拟机找不到/mnt/hgfs挂载目录

检测:
```
vmware-hgfsclient
```

安装:
```
apt-get install open-vm-dkms
apt-get install open-vm-tools-dkms
```

挂载:
```
mkdir /mnt/hgfs
chmod /mnt/hgfs

mount -t vmhgfs .host:/ /mnt/hgfs
vmhgfs-fuse .host:/ /mnt/hgfs
```

### 安装zsh

```
sudo apt-get update
sudo apt-get install zsh
chsh -s /bin/zsh
sudo vim /etc/passwd/
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

解决windows和linux文件结尾不一致
```
cd /Users/<your user>/.oh-my-zsh
find . -name "*.sh" | xargs dos2unix -f
find . -name "*.zsh" | xargs dos2unix -f
dos2unix -f themes/robbyrussel.zsh-theme
cd
dos2unix -f .zshrc
source ~/.zshrc
```
### pip超时

```
pip2 install maplotlib==2.2.3 --default-timeout=500
```

### 在python中生成GMT格式

```
from email.utils import formatdate
 
dt = formatdate(None, usegmt=True)
print(dt)
```

### markdownPad2在win10上HTML渲染组件错误
windows 10系统 需要下载 一个 awesomium_v1.6.6_sdk_win 这是一个 HTML UI ENGINE

下载地址 :
[http://markdownpad.com/download/awesomium_v1.6.6_sdk_win.exe](http://markdownpad.com/download/awesomium_v1.6.6_sdk_win.exe)