## Nextcloud 服务环境搭建过程

### 1. 创建用户
```shell script
groupadd www
useradd -r -g www www
```
### 2. 安装依赖
```shell script
yum -y install wget make gcc gcc-c++ pcre openssl openssl-devel zlib unzip cmake ncurses-devel libjpeg libjpeg-devel libpng libpng-devel libxml2 libxml2-devel curl-devel libtool libtool-ltdl libtool-ltdl-devel libevent libevent-devel zlib-static zlib-devel autoconf pcre-devel gd perl freetype freetype-devel bzip2 bzip2-devel gmp-devel libc-client-devel libicu-devel libzip-devel ImageMagick-devel libsmbclient-devel
```
> libzip-devel 这个依赖有版本要求

### 3. 安装Mysql8.0 - mysql-8.0.27-el7-x86_64.tar.gz
```shell script
tar -xvzf mysql-8.0.27-el7-x86_64.tar.gz
mv mysql-8.0.27-el7-x86_64 /usr/local/mysql
mkdir /usr/local/mysql/data
chown -R www:www /usr/local/mysql
chmod -R 755 /usr/local/mysql
touch /etc/profile.d/mysql.sh && echo 'export PATH=$PATH:/usr/local/mysql/bin' > /etc/profile.d/mysql.sh && source /etc/profile [设置环境变量]
mysqld --initialize --user=www --datadir=/usr/local/mysql/data --basedir=/usr/local/mysql [初始化数据库]
vim /etc/my.cnf
<<EOF
    [mysqld]
    #datadir=/var/lib/mysql
    socket=/tmp/mysql.sock
    # Disabling symbolic-links is recommended to prevent assorted security risks
    symbolic-links=0
    # Settings user and group are ignored when systemd is used.
    # If you need to run mysqld under a different user or group,
    # customize your systemd unit file for mariadb according to the
    # instructions in http://fedoraproject.org/wiki/Systemd
    #
    
    datadir = /usr/local/mysql/data
    port = 3306
    #sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
    max_connections = 600
    pid-file = /usr/local/mysql/mysql.pid
    
    character_set_server = utf8mb4
    collation_server = utf8mb4_general_ci
    #设置事务为读已提交
    transaction_isolation = READ-COMMITTED
    #设置binlog日志格式
    binlog_format = ROW
    innodb_file_per_table = 1
    
    [mysqld_safe]
    #log-error=/var/log/mariadb/mariadb.log
    #pid-file=/var/run/mariadb/mariadb.pid
    
    log-error = /usr/local/mysql/error.log
    pid-file = /usr/local/mysql/mysql.pid
    user = www
    tmpdir = /tmp
    
    [client]
    default-character-set = utf8mb4
    
    [server]
    skip_name_resolve = 1
    innodb_buffer_pool_size = 128M
    innodb_buffer_pool_instances = 1
    innodb_flush_log_at_trx_commit = 2
    innodb_log_buffer_size = 32M
    innodb_max_dirty_pages_pct = 90
    tmp_table_size = 64M
    max_heap_table_size = 64M
    slow_query_log = 1
    slow_query_log_file = /usr/local/mysql/slow.log
    long_query_time = 1
    
    #
    # include all files from the config directory
    #
    !includedir /etc/my.cnf.d
EOF
cp /usr/local/mysql8/support-files/mysql.server /etc/init.d/mysql
/etc/init.d/mysql start
//todo 利用 临时密码登录mysql,修改root密码
#开机启动 mysql
chmod +x /etc/init.d/mysql
chkconfig --add mysql
```
### 3. 安装PHP8
```shell script
./configure --prefix=/usr/local/php --with-config-file-path=/usr/local/php/etc --with-fpm-user=www --with-fpm-group=www --with-curl --enable-gd  --with-freetype --enable-mbstring --with-openssl --with-zip --with-zlib --with-pdo-mysql --with-bz2 --enable-intl --with-ldap --enable-ftp --with-imap --with-imap-ssl --enable-bcmath --with-gmp --enable-exif --with-kerberos --enable-fpm --enable-pcntl --enable-phar --with-jpeg --with-sodium --enable-exif
#如果遇到安装了libzip，此时还报libzip的错,则执行下面命令，然后再执行上一步
export PKG_CONFIG_PATH="/usr/local/lib/pkgconfig/"
make && make install
#如果在编译时遇到 error adding symbols: DSO missing from command，则在Makefile文件的EXTRA_LIBS这一行末尾添加 -llber，然后再次make
cp php.ini-x /usr/local/php/etc/php.ini
# 生成php-fpm.conf 以及 www.conf
#在网站的php-fpm.conf下（www.conf）取消注释一下几行
;env[HOSTNAME] = $HOSTNAME
;env[PATH] = /usr/local/bin:/usr/bin/:/bin
;env[TMP] = /tmp
;env[TMPDIR] = /tmp
;env[TEMP] = /tmp
vim /etc/systemed/system/php-fpm.service
<<EOF
    [Unit]
    Description=The php fastcgi process manager
    After=syslog.target network.target
    
    [Service]
    Type=simple
    PIDFile=/run/php-fpm.pid
    ExecStart=/usr/local/php/sbin/php-fpm --nodaemonize --fpm-config /usr/local/php/etc/php-fpm.conf
    ExecReload=/bin/kill -USR2 $MAINPID
    ExecStop=/bin/kill -SIGINT $MAINPID
    
    [Install]
    WantedBy=multi-user.target
EOF
touch /etc/profile.d/php.sh && echo 'export PATH=$PATH:/usr/local/php/bin' > /etc/profile.d/php.sh && source /etc/profile [设置环境变量]
systemctl daemon-reload
systemctl enable php-fpm.service #开机启动
# 安装PHP扩展:imagick、smbclient、redis
配置相应的配置
```

### 4. 安装nginx
```shell script
vim /etc/systemd/system/nginx.service
<<EOF
[Unit]
Description=nginx service
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable nginx.service #开机启动
touch /etc/profile.d/nginx.sh && echo 'export PATH=$PATH:/usr/local/nginx/sbin' > /etc/profile.d/nginx.sh && source /etc/profile [设置环境变量]
```
* [配置站点](nginx_conf.md ':include')
### 5. 安装Redis
```shell script
make && mv xx /usr/local/redis
touch /etc/profile.d/redis.sh && echo 'export PATH=$PATH:/usr/local/redis/src' > /etc/profile.d/redis.sh && source /etc/profile [设置环境变量]
vim /etc/init.d/redis
<<EOF
#!/bin/sh
#Configurations injected by install_server below....

EXEC=/usr/local/redis/src/redis-server
CLIEXEC=/usr/local/redis/src/redis-cli
PIDFILE=/var/run/redis_6379.pid
CONF="/usr/local/redis/redis.conf"
REDISPORT="6379"
###############
# SysV Init Information
# chkconfig: - 58 74
# description: redis_6379 is the redis daemon.
### BEGIN INIT INFO
# Provides: redis_6379
# Required-Start: $network $local_fs $remote_fs
# Required-Stop: $network $local_fs $remote_fs
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Should-Start: $syslog $named
# Should-Stop: $syslog $named
# Short-Description: start and stop redis_6379
# Description: Redis daemon
### END INIT INFO

case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
            echo "$PIDFILE exists, process is already running or crashed"
        else
            echo "Starting Redis server..."
            $EXEC $CONF
        fi
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
            echo "$PIDFILE does not exist, process is not running"
        else
            PID=$(cat $PIDFILE)
            echo "Stopping ..."
            $CLIEXEC -p $REDISPORT shutdown
            while [ -x /proc/${PID} ]
            do
                echo "Waiting for Redis to shutdown ..."
                sleep 1
            done
            echo "Redis stopped"
        fi
        ;;
    status)
        PID=$(cat $PIDFILE)
        if [ ! -x /proc/${PID} ]
        then
            echo 'Redis is not running'
        else
            echo "Redis is running ($PID)"
        fi
        ;;
    restart)
        $0 stop
        $0 start
        ;;
    *)
        echo "Please use start, stop, restart or status as first argument"
        ;;
esac
EOF
chmod +x /etc/init.d/redis
chkconfig --add redis
```
### 6.部署nextcloud代码
* nginx、apache+fpm 与 **nextcloud22.2.0**版本不兼容，有BUG，所以代码用了**21.0.5**版本

### 7.[SELinux配置](https://docs.nextcloud.com/server/21/admin_manual/installation/selinux_configuration.html)
### 8.[服务器调优](https://docs.nextcloud.com/server/21/admin_manual/installation/server_tuning.html)
### 9.[PHP配置内容缓存](https://docs.nextcloud.com/server/21/admin_manual/configuration_server/caching_configuration.html)

### 10. 注意事项
* nginx 、php-fpm 、nextcloud代码所在目录都必须是统一用户，否则会出现权限不足问题
* SELinux 配置错误，也有可能出现权限不足的问题
* redis 配置文件锁定时，也需要将redis启动用户加入到web服务的用户所在的用户组里，否则也会出现无法锁定问题
