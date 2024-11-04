# PHP & WordPress 开发环境搭建指南

## 1. 项目结构
首先创建以下项目结构：
```
project/
├── docker/
│   ├── php/
│   │   ├── Dockerfile
│   │   └── custom.ini
│   ├── wordpress/
│   │   └── Dockerfile
│   └── mysql/
│       ├── my.cnf
│       └── init.sql
├── php-src/
│   └── index.php
├── wp-content/
│   ├── plugins/
│   └── themes/
├── setup.sh
└── docker-compose.yml
```

## 2. 创建部署脚本 (setup.sh)

创建一个名为 `setup.sh` 的文件，包含以下内容：
```bash
#!/bin/bash

# 创建目录结构
mkdir -p project/{php-src,wp-content/{plugins,themes},docker/{php,wordpress,mysql}}

# 进入项目目录
cd project

# 创建 PHP 测试文件
cat > php-src/index.php << 'EOL'
<?php
phpinfo();
EOL

# 创建 docker-compose.yml
cat > docker-compose.yml << 'EOL'
version: '3'

services:
  php:
    build: 
      context: ./docker/php
      args:
        BUILDPLATFORM: linux/arm64
    ports:
      - "8001:80"
    volumes:
      - ./php-src:/var/www/html
      - ./docker/php/custom.ini:/usr/local/etc/php/conf.d/custom.ini
    environment:
      PHP_IDE_CONFIG: serverName=php-docker
      XDEBUG_MODE: debug
      XDEBUG_CONFIG: client_host=host.docker.internal client_port=9003
    extra_hosts:
      - "host.docker.internal:host-gateway"

  wordpress:
    depends_on:
      - db
    build:
      context: ./docker/wordpress
      args:
        BUILDPLATFORM: linux/arm64
    ports:
      - "8000:80"
    volumes:
      - wordpress_data:/var/www/html
      - ./wp-content:/var/www/html/wp-content
      - ./docker/php/custom.ini:/usr/local/etc/php/conf.d/custom.ini
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_TABLE_PREFIX: wp_
      WORDPRESS_DEBUG: 1
      XDEBUG_MODE: debug
      XDEBUG_CONFIG: client_host=host.docker.internal client_port=9003
    extra_hosts:
      - "host.docker.internal:host-gateway"

  db:
    image: mysql:8.0
    platform: linux/arm64
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/custom.cnf
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    command: '--default-authentication-plugin=mysql_native_password'
    cap_add:
      - SYS_NICE

  phpmyadmin:
    image: arm64v8/phpmyadmin:latest
    platform: linux/arm64
    ports:
      - "8080:80"
    environment:
      PMA_HOST: db
      MYSQL_ROOT_PASSWORD: root
    depends_on:
      - db

volumes:
  db_data: {}
  wordpress_data: {}
EOL

# 创建 PHP Dockerfile
cat > docker/php/Dockerfile << 'EOL'
ARG BUILDPLATFORM=linux/arm64
FROM --platform=$BUILDPLATFORM php:8.2-apache

RUN apt-get update && apt-get install -y \
    libzip-dev \
    zip \
    unzip \
    git \
    && docker-php-ext-install zip pdo pdo_mysql mysqli

COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

RUN pecl install xdebug \
    && docker-php-ext-enable xdebug

RUN a2enmod rewrite

RUN apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /var/www/html
EOL

# 创建 WordPress Dockerfile
cat > docker/wordpress/Dockerfile << 'EOL'
ARG BUILDPLATFORM=linux/arm64
FROM --platform=$BUILDPLATFORM wordpress:latest

RUN pecl install xdebug \
    && docker-php-ext-enable xdebug

RUN docker-php-ext-install pdo pdo_mysql

RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x wp-cli.phar \
    && mv wp-cli.phar /usr/local/bin/wp

WORKDIR /var/www/html
EOL

# 创建 PHP 配置文件
cat > docker/php/custom.ini << 'EOL'
[PHP]
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
date.timezone = Asia/Shanghai

[xdebug]
zend_extension = xdebug.so
xdebug.mode = debug
xdebug.start_with_request = yes
xdebug.client_host = host.docker.internal
xdebug.client_port = 9003
xdebug.idekey = PHPSTORM

display_errors = On
display_startup_errors = On
error_reporting = E_ALL
EOL

# 创建 MySQL 配置文件
cat > docker/mysql/my.cnf << 'EOL'
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
default-authentication-plugin = mysql_native_password
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION

[client]
default-character-set = utf8mb4
EOL

# 赋予脚本执行权限
chmod +x setup.sh

echo "项目结构创建完成！"
```

## 3. 使用说明

### 环境需求
- Docker Desktop for Mac (支持 ARM64)
- Git (可选)
- 终端访问权限

### 安装步骤

1. 创建项目目录：
```bash
mkdir my-dev-environment && cd my-dev-environment
```

2. 复制上述 setup.sh 的内容到新文件：
```bash
nano setup.sh  # 或使用其他编辑器
```

3. 赋予执行权限并运行：
```bash
chmod +x setup.sh
./setup.sh
```

4. 构建并启动容器：
```bash
docker-compose build --no-cache
docker-compose up -d
```

### 访问地址
- WordPress: http://localhost:8000
- PHP 开发环境: http://localhost:8001
- phpMyAdmin: http://localhost:8080

### 数据库访问信息

1. phpMyAdmin 登录：
- Root 用户:
  ```
  用户名：root
  密码：root
  ```
- WordPress 用户:
  ```
  用户名：wordpress
  密码：wordpress
  ```

2. MySQL 直接连接：
```
主机：localhost
端口：3306
数据库：wordpress
用户名：wordpress
密码：wordpress
```

### 常用操作命令

1. 启动所有服务：
```bash
docker-compose up -d
```

2. 停止所有服务：
```bash
docker-compose down
```

3. 查看服务状态：
```bash
docker-compose ps
```

4. 查看服务日志：
```bash
docker-compose logs -f [服务名]  # 服务名可以是 php, wordpress, db, phpmyadmin
```

5. 进入容器：
```bash
# PHP 容器
docker-compose exec php bash

# WordPress 容器
docker-compose exec wordpress bash

# MySQL 容器
docker-compose exec db bash
```

6. 重启特定服务：
```bash
docker-compose restart [服务名]
```

### 开发相关

1. PHP 开发：
- 代码位置：`./php-src/`
- URL: http://localhost:8001

2. WordPress 开发：
- 主题开发：`./wp-content/themes/`
- 插件开发：`./wp-content/plugins/`
- URL: http://localhost:8000

3. 数据库管理：
- phpMyAdmin: http://localhost:8080
- 直接连接 MySQL: localhost:3306

### 注意事项

1. 文件权限：
- 如果遇到权限问题，可能需要调整目录权限：
```bash
chmod -R 755 wp-content
```

2. 性能优化：
- 在 Docker Desktop 中适当分配内存和 CPU
- 考虑使用 Docker 的 BuildKit 来优化构建过程

3. 安全建议：
- 在生产环境中修改所有默认密码
- 限制数据库用户权限
- 考虑启用 HTTPS

4. 故障排除：
- 如果服务无法启动，检查端口占用：
```bash
lsof -i :8000
lsof -i :8001
lsof -i :8080
lsof -i :3306
```

- 如果构建失败，尝试清理 Docker 缓存：
```bash
docker system prune -a
```

### 备份和恢复

1. 数据库备份：
```bash
docker-compose exec db mysqldump -u root -proot wordpress > backup.sql
```

2. 数据库恢复：
```bash
cat backup.sql | docker-compose exec -T db mysql -u root -proot wordpress
```
