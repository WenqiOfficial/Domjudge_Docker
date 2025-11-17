# Domjudge_Docker

一个用于快速部署 DOMjudge 的 Docker Compose 配置，包含：
- dom-mariadb：MariaDB 数据库
- dom-server：DOMjudge DOMserver
- dom-judge：Judgehost

注意：Judgehost 需要启用 Linux cgroups

## 快速开始

前置要求
- 已安装 Docker 与 Docker Compose。
- Linux 主机需启用 cgroups（Judgehost 运行所需）。

启动步骤
1. 启动数据库与 DOMserver
   ```
   docker compose up -d dom-mariadb dom-server
   ```
2. 获取初始口令（admin 与 judgehost API 密钥）
   ```
   # admin 密码
   docker exec -it dom-server cat /opt/domjudge/domserver/etc/initial_admin_password.secret

   # API 密钥
   docker exec -it dom-server cat /opt/domjudge/domserver/etc/restapi.secret
   ```
3. 访问 Web
   - 打开 http://localhost:88/
   - 使用 admin 账号登录，初始密码见第 2 步。
4. 启动 Judgehost
   ```
   docker compose up -d dom-judge
   ```

版本匹配
- 请确保 dom-server 与 dom-judge 使用相同版本（当前 compose：8.3.1）。

## 当前 Compose 配置

- Web 端口：主机 88 -> 容器 80（dom-server）
- 数据库（dom-mariadb）
  - MYSQL_USER=domjudge / MYSQL_PASSWORD=DomJudgeBuild2025FUN
  - MYSQL_DATABASE=domjudge
  - MYSQL_ROOT_PASSWORD=MARIAdbPASS2025FUN
- 时区：
  - dom-mariadb: TZ=Asia/Shanghai
  - dom-server/dom-judge: CONTAINER_TIMEZONE=Asia/Shanghai（可改）
- Judgehost
  - 需要 privileged 与 `/sys/fs/cgroup:/sys/fs/cgroup:ro` 只读挂载
  - 建议设置 DOMSERVER_BASEURL=http://dom-server
  - 设置 DAEMON_ID 唯一（多实例建议 0,1,2,...）
  - JUDGEDAEMON_PASSWORD 建议使用 dom-server 启动时输出的 API 密钥（restapi.secret）

## DOMserver 支持的环境变量

- CONTAINER_TIMEZONE（默认 Europe/Amsterdam）
- MYSQL_HOST（默认 mariadb，Compose 中应为 dom-mariadb）
- MYSQL_USER（默认 domjudge）
- MYSQL_PASSWORD（默认 domjudge）
- MYSQL_ROOT_PASSWORD（默认 domjudge）
- MYSQL_DATABASE（默认 domjudge）
- DJ_DB_INSTALL_BARE（默认 0，设 1 进行 bare-install）
- FPM_MAX_CHILDREN（默认 40）
- TRUSTED_PROXIES（默认空，逗号分隔 IP 列表）
- WEBAPP_BASEURL（默认 /，如 /domjudge）
- MYSQL_PASSWORD_FILE / MYSQL_ROOT_PASSWORD_FILE（从文件读取密码，适配 Compose secrets）

示例（通过 secrets 传递密码）
```
services:
  dom-server:
    image: domjudge/domserver:${DOMJUDGE_VERSION}
    secrets:
      - domjudge-mysql-pw
    environment:
      MYSQL_PASSWORD_FILE: /run/secrets/domjudge-mysql-pw
secrets:
  domjudge-mysql-pw:
    file: ./secrets/mysql_password.txt
```

## Judgehost 支持的环境变量

- CONTAINER_TIMEZONE（默认 Europe/Amsterdam）
- DOMSERVER_BASEURL（默认 http://domserver/，不要手动加 /api）
- JUDGEDAEMON_USERNAME（默认 judgehost）
- JUDGEDAEMON_PASSWORD（默认 password，可用 JUDGEDAEMON_PASSWORD_FILE）
- DAEMON_ID（默认 0）
- DOMJUDGE_CREATE_WRITABLE_TEMP_DIR（默认 0，>=6.1 可用）
- RUN_USER_UID_GID（默认 62860）

## 常用命令

进入容器
```
docker exec -it dom-server bash
```

查看日志（DOMserver）
```
docker exec -it dom-server nginx-access-log
docker exec -it dom-server nginx-error-log
docker exec -it dom-server symfony-log
```

重启服务（DOMserver）
```
docker exec -it dom-server supervisorctl restart nginx
docker exec -it dom-server supervisorctl restart php
```