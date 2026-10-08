# LDAP 认证
如需配置 LDAP 认证，可按以下步骤操作。

操作步骤：修改服务端 `config.properties` 中的 LDAP 配置，重启服务。
```
login.auth.type=ldap

login.auth.password.encrypt=base64

ldap.server.url=ldap://192.168.1.99:389/

ldap.user.dn=cn={0},ou=ts,dc=neatlogic,dc=com
```
