# Docker 搭建 Nginx + php-fpm + mysql
>Author：邹智

>PS: 全文中有4处粗体，是我自己踩过的坑。希望对大家有帮助。

## 创建目录

1. 创建项目目录 `mkdir project`
2. 进入project目录 `cd project` 
3. 创建Nginx目录 `mkdir nginx`
4. 创建php-fpm目录 `mkdir php-fpm`
5. 创建mysql文件存储目录 `mkdir data`

## 配置Nginx
- 创建Dockerfile文件
- `fastcgi_pass my-app:9000;` **my-app是在docker-compose.yml中设定的services(php-fpm)名称**

```
#引用nginx镜像
FROM nginx
#将本地的配置文件复制到容器中。如果配置文件需要经常改动，建议volume本地文件。
COPY nginx.conf /etc/nginx/nginx.conf
#暴露80端口
EXPOSE 80
```
- 创建nginx.conf文件

```
#用户
user nginx nginx;
#nginx进程数
worker_processes 2;

#工作模式及连接数上限
events
{
    use epoll;
    worker_connections 1024;
}

#网络设置
http
{
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    server {
      listen          80; 
      index           index.php index.html;
      server_name     _;  
      root            /var/www/html/public;

      location / { 
            try_files $uri $uri/ /index.php$is_args$args;
        }   
    
      location ~ \.php$ {
          fastcgi_split_path_info ^(.+\.php)(/.+)$;
          #my-app是在docker-compose.yml中设定的php-fpm名称
          fastcgi_pass my-app:9000; 
          fastcgi_index index.php;
          include fastcgi_params;
          fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
          fastcgi_param PATH_INFO $fastcgi_path_info;
      }   
    }   
}
```

## 配置php-fpm
- 去 [Dockerhub官方php镜像](https://hub.docker.com/_/php) 选择你需要的php-fpm版本，并且下载相关的所有文件。**通常包括Dcokerfile、docker-php-ext-configure、docker-php-ext-install、docker-php-entrypoint、docker-php-ext-enable、docker-php-source 这六个文件。**

- Dcokerfile文件

```
FROM debian:jessie

# prevent Debian's PHP packages from being installed
# https://github.com/docker-library/php/pull/542
RUN set -eux; \
	{ \
		echo 'Package: php*'; \
		echo 'Pin: release *'; \
		echo 'Pin-Priority: -1'; \
	} > /etc/apt/preferences.d/no-debian-php

# dependencies required for running "phpize"
# (see persistent deps below)
ENV PHPIZE_DEPS \
		autoconf \
		dpkg-dev \
		file \
		g++ \
		gcc \
		libc-dev \
		make \
		pkg-config \
		re2c

# persistent / runtime deps
RUN apt-get update && apt-get install -y \
		$PHPIZE_DEPS \
		ca-certificates \
		curl \
		xz-utils \
	--no-install-recommends && rm -r /var/lib/apt/lists/*

ENV PHP_INI_DIR /usr/local/etc/php
RUN set -eux; \
	mkdir -p "$PHP_INI_DIR/conf.d"; \
# allow running as an arbitrary user (https://github.com/docker-library/php/issues/743)
	[ ! -d /var/www/html ]; \
	mkdir -p /var/www/html; \
	chown www-data:www-data /var/www/html; \
	chmod 777 /var/www/html

##<autogenerated>##
ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data --disable-cgi
##</autogenerated>##

# Apply stack smash protection to functions using local buffers and alloca()
# Make PHP's main executable position-independent (improves ASLR security mechanism, and has no performance impact on x86_64)
# Enable optimization (-O2)
# Enable linker optimization (this sorts the hash buckets to improve cache locality, and is non-default)
# Adds GNU HASH segments to generated executables (this is used if present, and is much faster than sysv hash; in this configuration, sysv hash is also generated)
# https://github.com/docker-library/php/issues/272
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"

ENV GPG_KEYS A917B1ECDA84AEC2B568FED6F50ABC807BD5DCD0 528995BFEDFBA7191D46839EF9BA0ADA31CBD89E 1729F83938DA44E27BA0F4D3DBDB397470D12172

ENV PHP_VERSION 7.1.27
ENV PHP_URL="https://secure.php.net/get/php-7.1.27.tar.xz/from/this/mirror" PHP_ASC_URL="https://secure.php.net/get/php-7.1.27.tar.xz.asc/from/this/mirror"
ENV PHP_SHA256="25672a3a6060eff37c865a0c84e284da50b7ee8cd57174c78f0ae244b90a96a8" PHP_MD5=""

RUN set -xe; \
	\
	fetchDeps=' \
		wget \
	'; \
	if ! command -v gpg > /dev/null; then \
		fetchDeps="$fetchDeps \
			dirmngr \
			gnupg \
		"; \
	fi; \
	apt-get update; \
	apt-get install -y --no-install-recommends $fetchDeps; \
	rm -rf /var/lib/apt/lists/*; \
	\
	mkdir -p /usr/src; \
	cd /usr/src; \
	\
	wget -O php.tar.xz "$PHP_URL"; \
	\
	if [ -n "$PHP_SHA256" ]; then \
		echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
	fi; \
	if [ -n "$PHP_MD5" ]; then \
		echo "$PHP_MD5 *php.tar.xz" | md5sum -c -; \
	fi; \
	\
	if [ -n "$PHP_ASC_URL" ]; then \
		wget -O php.tar.xz.asc "$PHP_ASC_URL"; \
		export GNUPGHOME="$(mktemp -d)"; \
		for key in $GPG_KEYS; do \
			gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
		done; \
		gpg --batch --verify php.tar.xz.asc php.tar.xz; \
		command -v gpgconf > /dev/null && gpgconf --kill all; \
		rm -rf "$GNUPGHOME"; \
	fi; \
	\
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $fetchDeps

COPY docker-php-source /usr/local/bin/

RUN set -eux; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libcurl4-openssl-dev \
		libedit-dev \
		libsqlite3-dev \
		libssl-dev \
		libxml2-dev \
		zlib1g-dev \
		${PHP_EXTRA_BUILD_DEPS:-} \
	; \
	rm -rf /var/lib/apt/lists/*; \
	\
	export \
		CFLAGS="$PHP_CFLAGS" \
		CPPFLAGS="$PHP_CPPFLAGS" \
		LDFLAGS="$PHP_LDFLAGS" \
	; \
	docker-php-source extract; \
	cd /usr/src/php; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
# https://bugs.php.net/bug.php?id=74125
	if [ ! -d /usr/include/curl ]; then \
		ln -sT "/usr/include/$debMultiarch/curl" /usr/local/include/curl; \
	fi; \
	./configure \
		--build="$gnuArch" \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
		\
# make sure invalid --configure-flags are fatal errors intead of just warnings
		--enable-option-checking=fatal \
		\
# https://github.com/docker-library/php/issues/439
		--with-mhash \
		\
# --enable-ftp is included here because ftp_ssl_connect() needs ftp to be compiled statically (see https://github.com/docker-library/php/issues/236)
		--enable-ftp \
# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
		--enable-mbstring \
# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
		--enable-mysqlnd \
		\
		--with-curl \
		--with-libedit \
		--with-openssl \
		--with-zlib \
		\
# bundled pcre does not support JIT on s390x
# https://manpages.debian.org/stretch/libpcre3-dev/pcrejit.3.en.html#AVAILABILITY_OF_JIT_SUPPORT
		$(test "$gnuArch" = 's390x-linux-gnu' && echo '--without-pcre-jit') \
		--with-libdir="lib/$debMultiarch" \
		\
		${PHP_EXTRA_CONFIGURE_ARGS:-} \
	; \
	make -j "$(nproc)"; \
	find -type f -name '*.a' -delete; \
	make install; \
	find /usr/local/bin /usr/local/sbin -type f -executable -exec strip --strip-all '{}' + || true; \
	make clean; \
	\
# https://github.com/docker-library/php/issues/692 (copy default example "php.ini" files somewhere easily discoverable)
	cp -v php.ini-* "$PHP_INI_DIR/"; \
	\
	cd /; \
	docker-php-source delete; \
	\
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
	find /usr/local -type f -executable -exec ldd '{}' ';' \
		| awk '/=>/ { print $(NF-1) }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	\
	php --version; \
	\
# https://github.com/docker-library/php/issues/443
	pecl update-channels; \
	rm -rf /tmp/pear ~/.pearrc

COPY docker-php-ext-* docker-php-entrypoint /usr/local/bin/

ENTRYPOINT ["docker-php-entrypoint"]
##<autogenerated>##
WORKDIR /var/www/html

RUN set -ex \
	&& cd /usr/local/etc \
	&& if [ -d php-fpm.d ]; then \
		# for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf"
		sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
		cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
	else \
		# PHP 5.x doesn't use "include=" by default, so we'll create our own simple config that mimics PHP 7+ for consistency
		mkdir php-fpm.d; \
		cp php-fpm.conf.default php-fpm.d/www.conf; \
		{ \
			echo '[global]'; \
			echo 'include=etc/php-fpm.d/*.conf'; \
		} | tee php-fpm.conf; \
	fi \
	&& { \
		echo '[global]'; \
		echo 'error_log = /proc/self/fd/2'; \
		echo; \
		echo '[www]'; \
		echo '; if we send this to /proc/self/fd/1, it never appears'; \
		echo 'access.log = /proc/self/fd/2'; \
		echo; \
		echo 'clear_env = no'; \
		echo; \
		echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
		echo 'catch_workers_output = yes'; \
	} | tee php-fpm.d/docker.conf \
	&& { \
		echo '[global]'; \
		echo 'daemonize = no'; \
		echo; \
		echo '[www]'; \
		echo 'listen = 9000'; \
	} | tee php-fpm.d/zz-docker.conf

EXPOSE 9000
CMD ["php-fpm"]
```

- 将这些文件放入事先创建好的php-fpm文件夹中

## 编写docker-compose.yml文件
- **建议配置好一个服务先测试是否有问题，}$$\color{red}{再此基础上追加新的服务，这样就算有问题了，也比较容易定位。经验之谈。**

```
#需要指定文件版本
version: '3' 

#网络名称 docker会为这些容器分配一个网段
networks:
  interview:

#服务，可以理解为所需要的容器
services:
  my-app:
    #根据php-fpm文件夹下的Dockerfile创建容器
    build: php-fpm
    networks: 
       - interview
    #指定本地文件同步到容器中
    volumes:
      - /application:/var/www/html  

  nginx:
    build: nginx
    networks: 
      - interview
    #容器的80端口映射到本地的80端口
    ports:
      - "80:80"
    volumes:
      - /application:/var/www/html   
 
  db: 
    #直接从DockerHub上下载镜像文件创建容器
    image: "mysql:5.7.15"
    networks:
      - interview
    #指定配置 -- 在DcokerHub中搜索mysql有更详细的配置说明
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: zhihu
    volumes:
      - $PWD/data:/var/lib/mysql
``` 


## 启动并使用docker-compose
- 进入docker-compose.yml文件所在的文件夹 `cd project`
- 创建并启动所有容器 `docker-compose up -d` -d是指在后台运行 **(如果容器配置有所变动，在此之前先执行 `dcoker-compose build`)**
- 进入容器 `docker-compose exec $serivce_name bash` 其中 `$service_name` 是你docker-compose.yml文件中定义的services名称，并不是 `docker-compose ps` 查看到的名称。
- 停止/启动 `docker-compose stop / docker-compose start`
- 停止并删除容器 `docker-compose down`
- 水平扩展/负载均衡 `docker-compose up --scale my-app=3` 可以通过这种方式同时开启多个服务 配置也需要做相应的改动


