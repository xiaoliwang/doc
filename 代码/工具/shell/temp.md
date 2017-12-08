### 免密码登录

```shell
# 查看是否有 SSH keys
ls -al ~/.ssh # 是否存在 id_rsa 和 id_rsa.pub


# 如果不存在，创建密钥对
ssh-keygen -t rsa 4096 -C "email" # 生成 id_rsa 和 id_rsa.pub。并复制到 ~/.ssh 目录


# 拷贝公钥到远程服务器

# authorized_keys 文件存在的情况
cat ~/.ssh/id_rsa.pub | ssh username@example.com "cat - >> ~/.ssh/authorized_keys"
# authorized_keys 文件不存在的情况
scp ~/.ssh/id_rsa.pub username@example.com:~/.ssh/authorized_keys
# 两者都兼容的情况
ssh-copy-id username@example.com

# 为了安全性，需要修改 authorized_keys 的权限
chmod 0644 ~/.ssh/authorized_keys

# 系统开启 SELinux，需执行如下命令
restorecon -R -v /root/.ssh

```



### 别名快捷登录

```shell
vim ~/.ssh/config
```

按如下格式进行修改：

```yaml
Host alias
	HostName example.com
	User domainuser
```



### 自动补全

```shell
# ~/.bash_profile 最后追加
complete -W "$(echo `cat ~/.ssh/config | grep 'Host '| cut -f 2 -d ' '|uniq`;)" ssh
```



### shell（bash）自动补全

```shell
chsh -s /usr/local/bin/fish
chsh -s /bin/bash

# 如果没有添加到 shells
echo /usr/local/bin/fish | sudo tee -a /etc/shells
```



