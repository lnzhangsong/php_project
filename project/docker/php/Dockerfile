ARG BUILDPLATFORM=linux/arm64
FROM --platform=$BUILDPLATFORM php:8.2-apache

# 安装基础扩展
RUN apt-get update && apt-get install -y \
  libzip-dev \
  zip \
  unzip \
  git \
  && docker-php-ext-install zip pdo pdo_mysql mysqli

# 安装 Composer
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# 安装 Xdebug
RUN pecl install xdebug \
  && docker-php-ext-enable xdebug

# 配置 Apache
RUN a2enmod rewrite

# 清理
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /var/www/html
