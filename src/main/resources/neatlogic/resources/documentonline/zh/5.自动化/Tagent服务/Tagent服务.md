# 关于neatlogic-tagent
neatlogic-tagent用于部署在受管目标操作系统上，是一种可平滑替代主机连接协议的可选方式。Tagent具备以下特点：

1. 采用Perl语言开发，运行环境依赖要求极低
2. 支持常见的Windows、Linux、SUSE、AIX等操作系统
3. 对操作系统资源占用极少，资源消耗范围为：CPU ≤ 2%，内存 ≤ 200MB
4. 同一受管机器支持多用户安装
5. 与[neatlogic-runner](../../../neatlogic-runner)建立心跳连接，定期探测目标环境和服务可用性
6. 支持从[neatlogic-runner](../../../neatlogic-runner)注册、管理以及自动匹配管理网段下发执行
7. 支持在[neatlogic-web](../../../neatlogic-web)上查看日志、重启、修改配置、升级等操作

# 适用场景
neatlogic-tagent适用于以下场景：

1. Windows类机器
2. 机器账号密码经常变更
3. 机器上存在多账号，不想在平台上维护不同用户密码
4. 深度使用自动化运维，如资源安装交付

# Tagent安装
## 登录neatlogic平台，添加执行器组
在"系统配置"->"执行器组管理"页面添加Runner组，网段范围必须包含Tagent的IP地址。

> **注意**：0.0.0.0/0 表示匹配所有IP

![img.png](images/img.png)
![img.png](images/img1.png)

## 安装步骤
- 开通网络策略
- 获取安装包
- 在目标受管机器安装

### 开通网络策略
| 源IP | 目的IP | 目的端口 | 协议 | 备注 |
|------|--------|----------|------|------|
| neatlogic-tagent-client主机 | neatlogic-runner主机 | 8084/8888 | TCP | 8084注册端口/8888心跳端口 |
| neatlogic-runner主机 | neatlogic-tagent-client主机 | 3939 | TCP | 命令下发端口 |

### 获取安装包
neatlogic-tagent提供两种获取安装包的方式：

**方式一：基于源码打包**
基于[neatlogic-tagent-client](https://gitee.com/neat-logic/neatlogic-tagent-client)工程打包。源代码和资源都在该项目中，需要自行编译（社区版暂不提供编译教程，有兴趣可自行研究）。

```
说明：Windows安装包内嵌了Perl运行时依赖环境和7z工具。
```

**方式二：获取现成安装包**
从[neatlogic-tagent-client](https://gitee.com/neat-logic/neatlogic-tagent-client)获取现成安装包（请查看readme）。

```
#####安装包说明############
#neatlogic-runner 自带3个安装包
#Unix类：tagent.tar
#Windows 32位：tagent_windows_x32.zip
#Windows 64位：tagent_windows_x64.zip
```

**配置下载路径**
为便于后续服务器通过Runner服务链接下载Tagent安装包，获取安装包后可将其放到Runner配置文件application.properties中tagent.download.path配置声明的路径下。以下以Runner IP 192.168.0.21为例：

```
# 获取Unix安装包
# 格式: http://neatlogic-runner机器IP:8084/autoexecrunner/tagent/download/tagent.tar
# 示例
curl tagent.tar http://192.168.0.21:8084/autoexecrunner/tagent/download/tagent.tar

# 获取Windows 64位安装包
# 示例
curl tagent_windows_x64.zip http://192.168.0.21:8084/autoexecrunner/tagent/download/tagent_windows_x64.zip
```

### 安装
- Unix类操作系统建议以root用户安装，root安装的Agent会注册服务
- Windows操作系统需以管理员方式打开cmd窗口进行安装

#### 手动安装
**Linux | SUSE | AIX | Unix类安装**

```bash
# 登录目标受管机器，下载安装包，建议统一存放/opt目录
cd /opt
curl tagent.tar http://192.168.0.21:8084/autoexecrunner/tagent/download/tagent.tar

# 解压
tar -xvf tagent.tar

# 查看shell类型
echo $0  # AIX操作系统需注意，大多数默认是ksh

# 执行安装
# 参数说明：--serveraddr neatlogic-runner的访问地址  --tenant 租户名称
# shell类型是bash，以下以runner ip是192.168.0.21，租户是demo为例：
sh tagent/bin/setup.sh --action install --serveraddr http://192.168.0.21:8084 --tenant demo

# shell类型是ksh，以下以runner ip是192.168.0.21，租户是demo为例：
sh tagent/bin/setup.ksh --action install --serveraddr http://192.168.0.21:8084 --tenant demo

# 安装完检查 (3个进程)
ps -ef |grep tagent 

# 查看日志
less tagent/run/root/logs/tagent.log 

# 查看配置 
less tagent/run/root/conf/tagent.conf

# 启停
service tagent start/stop 
```

**Windows类安装**

1. 查看Windows操作系统位数，选择对应安装包
2. 登录目标受管机器，下载安装包并拷贝到C盘，建议统一存放C盘根目录
3. 以管理员权限打开cmd窗口，并切换到C盘目录
4. 进入tagent_windows_x64目录，执行：service-install.bat

示例：
```cmd
cd c:\tagent_windows_x64
service-install.bat
```

在受管机器Tagent安装完成后，查看日志如果提示注册成功，则可在 系统配置->[Tagent管理](../../100.系统配置/5.基础服务/Tagent管理.md) 页面检查Tagent状态。

![](images/tagent.png)

# 其它补充
## Tagent配置
进入/opt/tagent/run/root/conf目录，编辑tagent.conf文件，保存后需重启Tagent服务。关键参数说明：

| 参数 | 备注 | 是否必填 |
|------|------|----------|
| credential | 加密后的密码串 | 是 |
| listen.port | Tagent的端口 | 是 |
| proxy.group | Runner组IP:端口 | 否 |
| proxy.group.id | Runner组ID | 否 |
| proxy.registeraddress | Tagent在Runner的注册地址，需带上租户的UUID | 是 |
| tagent.id | Tagent ID | 否 |
| tenant | 租户UUID | 是 |

以安装在192.168.0.25的Tagent、192.168.0.21的Runner（服务端口为8084，心跳端口为8888）、192.168.0.25的neatlogic（租户为demo）为例：

```properties
credential={ENCRYPTED}19chdeh34c738cb575fef816607
exec.timeout=900
listen.addr=0.0.0.0
listen.backlog=16
listen.port=3939
proxy.group=
proxy.group.id=
proxy.registeraddress=http://192.168.0.21:8084/autoexecrunner/public/api/rest/tagent/register?tenant=demo
read.timeout=5
tagent.id=
tenant=demo
```

## Tagent重新注册

1. 停止Tagent服务
2. 在neatlogic系统中，打开"系统配置"->"Tagent管理"页面，删除Tagent
   ![](images/tagent_delete.png)
3. 还原tagent.conf文件
4. 重启Tagent服务

## Tagent卸载
**Linux | SUSE | AIX | Unix类服务卸载**

```bash
cd /opt 

# 查看Shell类型
echo $0

# Shell为bash
sh tagent/bin/setup.sh --action uninstall

# Shell为ksh
sh tagent/bin/setup.ksh --action uninstall

# 删除安装目录
rm -rf tagent
```

**Windows类服务卸载**

```cmd
# 以管理员权限打开cmd窗口，切换到Tagent安装目录
cd c:\tagent_windows_x64

# 执行卸载
service-uninstall.bat

# 删除安装目录
rd /s /q c:\tagent_windows_x64
```

## Tagent服务相关命令

**1. 启动Tagent服务命令**

```bash
service tagent start
# 或者
/bin/systemctl start tagent.service
```

**2. 停止Tagent服务命令**

```bash
service tagent stop
# 或者
/bin/systemctl stop tagent.service 
```
