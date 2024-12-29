# 拉取镜像

使用以下命令拉取 OpenLDAP 的官方镜像或社区维护的镜像：

`docker pull osixia/openldap:latest `

# 创建自定义网络

首先，您需要创建一个自定义的 Docker 网络。运行以下命令：

`docker network create ldap_network`

这将创建一个名为 `ldap_network` 的网络。

# 运行容器

可以使用以下命令快速启动一个 OpenLDAP 实例：

```
docker run -d 
  --name openldap 
  --hostname ldap.example.com 
  --network ldap_network 
  -p 389:389 -p 636:636 
  -e LDAP_ORGANISATION="Example Inc." 
  -e LDAP_DOMAIN="example.com" 
  -e LDAP_ADMIN_PASSWORD="admin_password" 
  osixia/openldap:latest
```

### 参数说明

* `--name openldap`: 容器名称。
* `--hostname ldap.example.com`: 设置容器的主机名。
* `-p 389:389`: 将本地 389 端口映射到容器内的 389 端口，用于明文 LDAP 连接。
* `-p 636:636`: 将本地 636 端口映射到容器内的 636 端口，用于 LDAPS 加密连接。
* `-e LDAP_ORGANISATION`: 设置 LDAP 的组织名称。
* `-e LDAP_DOMAIN`: 设置 LDAP 的域名。
* `-e LDAP_ADMIN_PASSWORD`: 设置管理员密码。

### Docker Compose（可选/待验证）

如果需要更复杂的配置或多个服务协同工作，可以使用 Docker Compose。

创建 `docker-compose.yml` 文件：

```
version: '3.8'
services:
  openldap:
    image: osixia/openldap:latest
    container_name: openldap
    hostname: ldap.example.com
    ports:
      - "389:389"
      - "636:636"
    environment:
      LDAP_ORGANISATION: "Example Inc."
      LDAP_DOMAIN: "example.com"
      LDAP_ADMIN_PASSWORD: "admin_password"
    volumes:
      - ./ldap-data:/var/lib/ldap
      - ./ldap-config:/etc/ldap/slapd.d
```

# 访问 OpenLDAP

### 使用 LDAP 客户端：

本地安装的客户端工具，例如 **LDAP Admin** 或  **Apache Directory Studio** 。

配置连接地址：`ldap://localhost:389` 或 `ldaps://localhost:636`。

管理员 DN: `cn=admin,dc=example,dc=com`。

管理员密码：`admin_password`。

### 使用命令行工具：

```
ldapsearch -x -H ldap://localhost:389 -D "cn=admin,dc=example,dc=com" -w admin_password -b "dc=example,dc=com"
```

# 持久化数据（待验证）

默认情况下，容器中的数据存储在内存中，停止后会丢失。如果需要持久化，请挂载数据卷：

* `/var/lib/ldap`: 数据目录。
* `/etc/ldap/slapd.d`: 配置目录。

例如：

```
docker run -d 
  --name openldap 
  -p 389:389 -p 636:636 
  -v $(pwd)/ldap-data:/var/lib/ldap 
  -v $(pwd)/ldap-config:/etc/ldap/slapd.d 
  -e LDAP_ORGANISATION="Example Inc." 
  -e LDAP_DOMAIN="example.com" 
  -e LDAP_ADMIN_PASSWORD="admin_password" 
  osixia/openldap:latest
```

# 扩展功能

如果需要配置 Web UI 管理工具，可以使用 [phpldapadmin](https://github.com/osixia/docker-phpLDAPadmin)。

例如：

```
docker run -d 
  --name phpldapadmin 
  --network ldap_network 
  -p 8080:80 
  -e PHPLDAPADMIN_LDAP_HOSTS=openldap 
  -e PHPLDAPADMIN_HTTPS=false 
  osixia/phpldapadmin:latest
```

```
Warning: This web connection is unencrypted.
这可能是因为您禁用了 HTTPS（使用了 PHPLDAPADMIN_HTTPS=false），如果您明确不需要加密通信（例如测试环境），可以安全忽略此警告。
```
