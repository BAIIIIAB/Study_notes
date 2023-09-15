# SSH

## SSH��¼
SSH������Զ�̵�½��������������Զ�̷��������߱��ط��������ɡ�

����
```shell
ssh user@hostname
```

���� `user` �û�����`hostname` IP��ַ������

��һ�ε�¼ʱ����ʾ��

The authenticity of host 'xxx.xx.xx.xxx (xxx.xx.xx.xxx)' can't be established.
ECDSA key fingerprint is SHA256:iy237yysfCe013/l+kpDGfEG9xxHxm0dnxnAbJTPpG8.
Are you sure you want to continue connecting (yes/no/[fingerprint])?

����yes��Ȼ��س����ɡ�
�����Ὣ�÷���������Ϣ��¼��~/.ssh/known_hosts�ļ��С�

Ȼ���������뼴�ɵ�¼��Զ�̷������С�

Ĭ�ϵ�¼�˿ں�Ϊ22��������¼ĳһ�ض��˿ڣ�

```shell
ssh user@hostname -p 22
```

��ssh�Ķ˿ڲ���22�Ļ�����config����Ӷ�Ӧ�Ķ˿ڼ���
����˿�Ϊ20000�����`Port 20000`

## ������¼
�����ļ�`~/.ssh/config`
Ȼ�����ļ������룺
```
Host myserver1
    HostName IP��ַ������
    User �û���
Host myserver2
    HostName IP��ַ������
    User �û��� 
```

֮����ʹ�÷�����ʱ������ֱ��ʹ�ñ���`myserver1`, `myserver2`

## ���ܵ�¼

������Կ��
`ssh-keygen`
Ȼ��һ·�س����ɡ�
ִ�н����� `~/.ssh/`Ŀ¼�»�������ļ���
- `id_rsa`��˽Կ�������׸����˿���
- `id_rsa.pub`����Կ

֮�������ܵ�¼�ĸ����������ͽ���Կ�����ĸ����������ɡ�
���磬�����ܵ�¼`myserver`���������򽫹�Կ�е����ݣ����Ƶ�`myserver`�е�`~/.ssh/authorized_keys`�ļ��Ｔ�ɡ�

Ҳ����ʹ����������һ����ӹ�Կ��
`ssh-copy-id myserver`

## ִ������

�����ʽ��`ssh user@hostname command`

���磺`ssh user@hostname ls -a`

```shell
# �������е�$i����������ֵ
ssh myserver 'for((i = 0; i < 10; i ++)) do echo $i; done'
```

����

```shell
# ˫�����е�$i������������ֵ
ssh myserver ��for((i = 0; i < 10; i ++)) do echo $i; done��
```

