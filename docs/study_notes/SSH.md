# SSH

## SSH登录
SSH可用来远程登陆服务器，不论是远程服务器或者本地服务器均可。

命令
```shell
ssh user@hostname
```

其中 `user` 用户名，`hostname` IP地址或域名

第一次登录时会提示：

The authenticity of host 'xxx.xx.xx.xxx (xxx.xx.xx.xxx)' can't be established.
ECDSA key fingerprint is SHA256:iy237yysfCe013/l+kpDGfEG9xxHxm0dnxnAbJTPpG8.
Are you sure you want to continue connecting (yes/no/[fingerprint])?

输入yes，然后回车即可。
这样会将该服务器的信息记录在~/.ssh/known_hosts文件中。

然后输入密码即可登录到远程服务器中。

默认登录端口号为22。如果想登录某一特定端口：

```shell
ssh user@hostname -p 22
```

若ssh的端口不是22的话，在config中添加对应的端口即可
例如端口为20000，添加`Port 20000`

## 别名登录
创建文件`~/.ssh/config`
然后在文件中输入：
```
Host myserver1
    HostName IP地址或域名
    User 用户名
Host myserver2
    HostName IP地址或域名
    User 用户名 
```

之后再使用服务器时，可以直接使用别名`myserver1`, `myserver2`

## 免密登录

创建密钥：
`ssh-keygen`
然后一路回车即可。
执行结束后， `~/.ssh/`目录下会多两个文件：
- `id_rsa`：私钥（别轻易给别人看）
- `id_rsa.pub`：公钥

之后想免密登录哪个服务器，就将公钥传给哪个服务器即可。
例如，想免密登录`myserver`服务器。则将公钥中的内容，复制到`myserver`中的`~/.ssh/authorized_keys`文件里即可。

也可以使用如下命令一键添加公钥：
`ssh-copy-id myserver`

## 执行命令

命令格式：`ssh user@hostname command`

例如：`ssh user@hostname ls -a`

```shell
# 单引号中的$i可以用来求值
ssh myserver 'for((i = 0; i < 10; i ++)) do echo $i; done'
```

或者

```shell
# 双引号中的$i不可以用来求值
ssh myserver “for((i = 0; i < 10; i ++)) do echo $i; done”
```

