# UBUNTU下安装nginx+php+mysql

## nginx

### 安装
sudo apt-get install nginx


### 配置日志轮询
sudo vim /root/shell/logcron.sh
```shell
#!/bin/bash
log_dir="/data/log/nginx"
time=`date +%Y%m%d`
/bin/mkdir ${log_dir}/${time}
/bin/mv  ${log_dir}/*.log ${log_dir}/${time}
kill -HUP `cat /run/nginx.pid`
#/usr/local/nginx/sbin/nginx -s reload
#find ${log_dir} -type f -mtime +15 -exec rm {} \;
find ${log_dir} -atime +15 | xargs rm -rf {}
```
sudo crontab -e
```shell
59 23 * * * /root/shell/logcron.sh > /dev/null 2>&1
```

### 修改配置
```shell
sudo sed -i 's#$request_filename;#$document_root$fastcgi_script_name;#' /etc/nginx/fastcgi_params
sudo sed -i 's#sendfile on;#sendfile on;\n\tclient_max_body_size 100m;#' /etc/nginx/nginx.conf
```

### 加一个默认的虚拟服务
sudo vim /etc/nginx/sites-available/default.conf
```
server {
        listen 80;
        root /data/www;
        index index.php index.html index.htm;
        server_name localhost;

        error_page 404 /404.html;
        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                include fastcgi_params;
        }
}
```

```
sudo rm /etc/nginx/sites-enabled/default
sudo ln -svf  /etc/nginx/sites-available/default.conf  /etc/nginx/sites-enabled/
```


## PHP

### 安装
sudo apt-get install php5 php5-cgi -y

### 配置
sudo vim /etc/init.d/php-fastcgi
```shell
#!/bin/bash
BIND=127.0.0.1:9000
USER=nobody
PHP_FCGI_CHILDREN=15
PHP_FCGI_MAX_REQUESTS=1000

PHP_CGI=/usr/bin/php-cgi
PHP_CGI_NAME=`basename $PHP_CGI`
PHP_CGI_ARGS="- USER=$USER PATH=/usr/bin PHP_FCGI_CHILDREN=$PHP_FCGI_CHILDREN PHP_FCGI_MAX_REQUESTS=$PHP_FCGI_MAX_REQUESTS $PHP_CGI -b $BIND"
RETVAL=0

start() {
      echo -n "Starting PHP FastCGI: "
      start-stop-daemon --quiet --start --background --chuid "$USER" --exec /usr/bin/env -- $PHP_CGI_ARGS
      RETVAL=$?
      echo "$PHP_CGI_NAME."
}
stop() {
      echo -n "Stopping PHP FastCGI: "
      killall -q -w -u $USER $PHP_CGI
      RETVAL=$?
      echo "$PHP_CGI_NAME."
}

case "$1" in
    start)
      start
  ;;
    stop)
      stop
  ;;
    restart)
      stop
      start
  ;;
    *)
      echo "Usage: php-fastcgi {start|stop|restart}"
      exit 1
  ;;
esac
exit $RETVAL
```

### 设置开机启动
```shell
sudo chmod +x /etc/init.d/php-fastcgi
sudo /etc/init.d/php-fastcgi start
sudo update-rc.d php-fastcgi defaults
```

### 配置PHP.ini文件
```shell
sudo sed -i 's#;open_basedir =#open_basedir = /data/www:/tmp#' /etc/php5/cgi/php.ini
sudo sed -i 's#post_max_size = 8M#post_max_size = 100M#' /etc/php5/cgi/php.ini
sudo sed -i 's#upload_max_filesize = 2M#upload_max_filesize = 100M#' /etc/php5/cgi/php.ini
sudo sed -i 's#max_execution_time = 30#max_execution_time = 120#' /etc/php5/cgi/php.ini
sudo sed -i 's#;short_open_tag = Off#short_open_tag = On#' /etc/php5/cgi/php.ini
sudo sed -i 's#short_open_tag = Off#short_open_tag = On#' /etc/php5/cgi/php.ini
sudo sed -i 's#session.gc_probability = 0#session.gc_probability = 1#' /etc/php5/cgi/php.ini
```

## 按需求安装相关PHP组件
```shell
GD图片处理库
sudo apt-get install php5-gd -y

php-mysql支持
sudo apt-get install php5-mysql -y

php5-curl支持
sudo apt-get install php5-curl curl -y

php5-redis支持
sudo apt-get install php5-redis -y
```

## mysql 
sudo apt-get install mysql-server-5.6 -y

sudo sed -i 's#bind-address#\#bind-address#' /etc/mysql/my.cnf

sudo vim /etc/mysql/conf.d/my-ext.cnf
```shell
[mysqld]
#* base setting
skip-name-resolve
max_connections         = 1024 #8192
explicit_defaults_for_timestamp = TRUE
datadir                 = /var/lib/mysql

key_buffer              = 64
myisam_sort_buffer_size = 8M #64M
concurrent_insert       = 2
delayed_insert_timeout  = 300
query_cache_limit       = 1M
query_cache_size        = 16M
table_open_cache        = 65536

# * log and slow queries
log_error               = /var/log/mysql/error.log
slow-query-log          = 1
slow-query-log-file     = /var/log/mysql/mysql-slow.log
long_query_time         = 1


# * binlog
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_cache_size       = 32K
max_binlog_cache_size   = 512M #2G
expire_logs_days        = 10
max_binlog_size         = 512M
#binlog_do_db           = include_database_name
#binlog_ignore_db       = include_database_name


# * Security Features
safe-user-create
local-infile            = 0


# * Innodb
innodb_file_per_table
innodb_buffer_pool_size         = 128M #1G
innodb_buffer_pool_instances    = 2 #8
innodb_log_files_in_group       = 4
innodb_log_file_size            = 1G
innodb_log_buffer_size          = 8M #64M
innodb_flush_log_at_trx_commit  = 1
```

重启服务并测试
```shell
/etc/init.d/php-fastcgi restart
/etc/init.d/mysql restart
nginx -s reload
```


