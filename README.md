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


