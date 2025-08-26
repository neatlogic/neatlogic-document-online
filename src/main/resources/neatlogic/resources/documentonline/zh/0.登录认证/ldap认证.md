# ldap认证
neatlogic服务的config.properties 修改以下配置：
```
login.auth.type=ldap

login.auth.password.encrypt=base64

ldap.server.url=ldap://192.168.1.99:389/

ldap.user.dn=cn={0},ou=ts,dc=neatlogic,dc=com
```