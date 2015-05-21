title: "GitHub上部署Hexo网站"
date: 2015-04-20 08:57:24
tags: [Hexo]
---
Hexo建站参见[build-site-record-hexo]( http://uuangian.github.io/2015/04/17/build-site-record-hexo/)

## 建立远程GitHub连接

### 本地Git客户端

   在[Hexo建站](http://uuangian.github.io/2015/04/17/build-site-record-hexo/)中提到Windows下安装Git客户端msysgit，安装完成后即可使用Git Bash进命令操作


### 生成新的SSH Key
在GitBash中
```
ssh-keygen -t rsa -C "邮件地址@youremail.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/your_user_directory/.ssh/id_rsa):
```
在GitHub中添加生成的SSH Key
参见[add-your-ssh-key-to-your-account](https://help.github.com/articles/generating-ssh-keys/#step-4-add-your-ssh-key-to-your-account)

最后验证:
```
ssh -T git@github.com
```
详细设置可以参考:
[GitHub Help - Generating SSH Keys](https://help.github.com/articles/generating-ssh-keys/)

***遇到的报错处理***
 -  Could not open a connection to your authentication agent 错误, win7 下 使用
 ```
 eval $(ssh-agent)
 ```
 -  在ssh -vT git@github.com 报 port 22: Attempt to connect timed out without establishing a connection
     
     需要 编写config文件在.ssh/目录下
```
Host github.com
User git
Hostname ssh.github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa
Port 443
```
