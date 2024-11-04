ARG BUILDPLATFORM=linux/arm64
FROM --platform=$BUILDPLATFORM wordpress:latest

# 安装 Xdebug
RUN pecl install xdebug \
  && docker-php-ext-enable xdebug

# 安装额外的 PHP 扩展
RUN docker-php-ext-install pdo pdo_mysql

# 安装 WP-CLI
RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
  && chmod +x wp-cli.phar \
  && mv wp-cli.phar /usr/local/bin/wp

WORKDIR /var/www/html
