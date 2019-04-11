---
title: Wazuh邮件报警配置总结
date: 2019-04-05 15:56:38
tags: Wazuh-HIDS
---

#### 背景
Wazuh 的 OSSEC提供邮件报警功能，配置部分在

```
# vim /var/ossec/etc/ossec.conf
```

```
<global>
    <jsonout_output>yes</jsonout_output>
    <alerts_log>yes</alerts_log>
    <logall>no</logall>
    <logall_json>no</logall_json>
    <email_notification>yes</email_notification>
    <smtp_server>localhost</smtp_server>
    <email_from>aaa@163.com</email_from>
    <email_to>123@qq.com</email_to>
    <email_maxperhour>12</email_maxperhour>
    <queue_size>131072</queue_size>
  </global>

```
但是只提供了smtp_server的选项，并没有提供账号密码认证部分选项。
Wazuh官方文档也说到了：如果想用第三方邮件系统的话，需要自己在Wazuh的Server上安装一个邮件代理：比如 **postfix**。

这里以使用163邮箱作为发邮件邮箱为例。

操作系统为：centos.

#### 安装
安装流程采用Wazuh官方文档提供的安装方式。

1. 安装依赖包

```
# yum update && yum install postfix mailx cyrus-sasl cyrus-sasl-plain
```

2. 打开postfix的配置文件/etc/postfix/main.cf，添加如下代码：

```
relayhost = [smtp.163.com]:25
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_use_tls = no
sender_canonical_maps = hash:/etc/postfix/sender_canonical
```
解释一下：

*relayhost*: smtp服务器的域名和端口信息

*smtp_sasl_auth_enable*：是否需要权限验证

*smtp_sasl_password_maps*：发件人账号密码对应关系

*smtp_use_tls*：是否启用tls

*sender_canonical_maps*：发件人跟系统账户之间的对应关系，这个是重点


3. 然后配置发件人邮件地址和密码信息


```
# echo [smtp.163.com]:25 USERNAME@163.com:PASSWORD > /etc/postfix/sasl_passwd
# postmap /etc/postfix/sasl_passwd
# chmod 400 /etc/postfix/sasl_passwd
```


```
# chown root:root /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
# chmod 0600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```

4. 配置系统账号与发件人之间的对应关系

```
# touch /etc/postfix/sender_canonical
# echo liu.xxx   xxx@163.com > /etc/postfix/sender_canonical
# # postmap /etc/postfix/sender_canonical
```
注：这一步很重要，因为postfix默认会把系统用户名当成邮件发件人，比如把root@domain当做发件人。很多第三方邮箱是要求发件人姓名必须和发件人邮箱相同，否则会提示错误，发送失败。

5. 重启 Postfix

```
# systemctl reload postfix
```

6. 测试是否能收到邮件

```
# echo "Test mail from postfix" | mail -s "Test Postfix" you@example.com
```

7. 修改ossec配置

```
<global>
  <email_notification>yes</email_notification>
  <smtp_server>localhost</smtp_server>
  <email_from>USERNAME@gmail.com</email_from>
  <email_to>you@example.com</email_to>
</global>
```


#### 参考

https://documentation.wazuh.com/current/user-manual/manager/manual-email-report/smtp_authentication.html#smtp-authentication

http://www.cnblogs.com/tugeler/p/6620150.html