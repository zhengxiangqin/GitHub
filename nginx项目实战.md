# nginx项目实战
## 模拟场景
- **模拟4台Linux服务器，2台Nginx+2台LAP（发布两个网站WordPress+Discuz）+2台MYSQL（主从，可以共用），构建Nginx均衡2台LAMP架构，Nginx设置两个Server虚拟主机，分别访问WordPress网站和Discuz网站。**
### 环境准备
- nginx主机1:192.168.1.16
- nginx主机2:192.168.1.18
- httpd主机1+mysql（主）:192.168.1.11 
- httpd主机2+mysql（从）:192.168.1.12
**为避免对实验超成影响，关闭防火墙及SELIUNX:**
`systemctl stop firewalld`
`setenforce 0`
### 在httpd两台主机上安装服务及塔建mysql主从
#### 安装httpd、mariadb及相关组件
**在httpd两台主机上**
`yum install httpd httpd-devel php php-devel php-mysql mariadb mariadb-server mariadb-devel -y`
`systemctl start mariadb`
#### 塔建mysql主从
**在192.168.1.11上：**
`vim /etc/my.cnf`
```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
server_id = 1
log-bin = mysql-bin
```
`systemctl restart mariadb`
`mysql`
`grant replication slave on *.* to 'reuser'@'192.168.1.%' identified by '123456';`
`show master status\G;`
```
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 |      395 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)
```
**在192.168.1.12从库机上**
`vim /etc/my.cnf`
```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
server_id = 2
relay_log = relay-log
read_only = ON
```
`systemctl restart mairadb`
`mysql`
`change master to master_host='192.168.1.11',master_user='reuser',master_password='123456',master_port=3306,master_log_file='mysql-bin.000003',master_log_pos=395;`
**在主库上**
`create database discuz charset=utf8;`
`create database wordpress charset=utf8;`
`grant all on discuz.* to discuz@'%' identified by '123456';`
`grant all on wordpress.* to wordpress@'%' identified by '123456';`
`flush privileges;`
**在从库上查看数据库是否同步过来**
`select user,host,password from mysql.user;`
```
| user      | host                  | password                                  |
+-----------+-----------------------+-------------------------------------------+
| root      | localhost             |                                           |
| root      | localhost.localdomain |                                           |
| root      | 127.0.0.1             |                                           |
| root      | ::1                   |                                           |
|           | localhost             |                                           |
|           | localhost.localdomain |                                           |
| wordpress | %                     | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
| reuser    | 192.168.1.%           | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
| discuz    | %                     | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
+-----------+-----------------------+-------------------------------------------+
9 rows in set (0.00 sec)
```
### 配置httpd主页
#### 在192.168.1.11主机上安装disxuz及wordpress
`cd /var/www/html`
`wget http://download.comsenz.com/DiscuzX/3.1/Discuz_X3.1_SC_UTF8.zip`
`wget  https://cn.wordpress.org/wordpress-4.9.4-zh_CN.tar.gz`
`mkdir discuz`
`unzip Discuz_X3.1_SC_UTF8.zip -d discuz/`  #将discuz解压到discuz目录下
`tar -xzf wordpress-4.9.4-zh_CN.tar.gz`
`cd discuz`
`mv upload/* .`
`cd /var/www/html/discuz/`
`chmod o+w config/ data/ uc_* -R` #授权
`vim /etc/hosts`
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.11 mysql.abc.net #数据库服务器名称
```
`cd /var/www/html/wordpress`
`cp wp-config-sample.php wp-config.php`
`vim wp-config.php`
```
/** WordPress数据库的名称 */
define('DB_NAME', 'wordpress');

/** MySQL数据库用户名 */
define('DB_USER', 'wordpress');

/** MySQL数据库密码 */
define('DB_PASSWORD', '123456');

/** MySQL主机 */
define('DB_HOST', 'mysql.abc.net');

/** 创建数据表时默认的文字编码 */
define('DB_CHARSET', 'utf8');
```
`cd /etc/httpd/conf`
`mkdir domains`
`cd domains/`
`vim dz.abc.net`
```
<VirtualHost *:80>
    ServerAdmin support@jfdu.net
    DocumentRoot "/var/www/html/discuz"
    ServerName dz.abc.net
    ErrorLog "logs/dz.abc.net_error_log"
    CustomLog "logs/dz.abc.net_access_log" common
</VirtualHost>
```
`vim wp.abc.net`
```
<VirtualHost *:80>
    ServerAdmin support@jfdu.net
    DocumentRoot "/var/www/html/wordpress"
    ServerName wp.abc.net
    ErrorLog "logs/wp.abc.net_error_log"
    CustomLog "logs/wp.abc.net_access_log" common
</VirtualHost>
```
`vim /etc/httpd/conf/httpd.conf`
```
Include conf.modules.d/*.conf
Include conf/domains/* #将domanins目录下的配置文件包含进来
```
`httpd -t` #检查httpd语法
`systemctl restart httpd`
#### 创建电脑hosts文件
**在C:\Windows\System32\drivers\etc路径下，新建文本文件，命名为hosts(无后缀），用记事本打开，输入：**
**192.168.1.11 dz.abc.net  wp.abc.net**
**在浏览器上分别输入dz.abc.net  　wp.abc.net 根据提示安装**
![](_v_images/20191013092753409_25772.png)
**至此，192.168.1.11主机完成安装**
#### 配置192.168.1.12主机
**12主机发布的站点一样，把11主机上的相关文件复制过去即可**
`scp -r /var/www/html/* root@192.168.1.12:/var/www/html/` #在11主上执行
`scp -r /etc/httpd/conf/* root@192.168.1.12:/etc/httpd/conf/`
`echo '192.168.1.11  mysql.abc.net' >> /etc/hosts`
`cd /var/www/html/discuz/`
`chmod o+w config/ data/ uc_* -R`
`systemctl restart httpd`
**将电脑hosts文件改为：**
`192.168.1.12  dz.abc.net  wp.abc.net`
**在浏览器中分别输入 dz.abc.net   wp.abc.net**
**12主机配置完成**
### 配置nginx主机做转发
`yum install -y nginx`
`vim /etc/nginx/nginx.conf`
```
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    include domains/*;
}
```
**为nginx配置虚拟主机**
`mkdir /etc/nginx/domains`
`vim /etc/nginx/domains/dz.abc.net`
```
upstream web_dz {
        server 192.168.1.11:80;
        server 192.168.1.12:80;
}
server {
        listen           80;
        server_name      dz.abc.net;
        location / {
            root   html;
            index  index.html index.htm;
            proxy_set_header  Host $host;
            proxy_pass  http://web_dz;
        }
        error_page  500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
       }
}
```
`vim /etc/nginx/domains/wp.abc.net`
```
upstream web_wp {
        server 192.168.1.11:80;
        server 192.168.1.12:80;
}
server {
        listen           80;
        server_name      wp.abc.net;
        location / {
            root   html;
            index  index.html index.htm;
            proxy_set_header  Host $host;
            proxy_pass  http://web_wp;  #web_wp要与upstream的名称一致
        }
        error_page  500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
       }
}
```
`nginx -t`
`nginx -s reload`
**修改电脑hosts文件**
`192.168.1.16  dz.abc.net  wp.abc.net`
**登陆浏览器，访问站点,成功访问**
**关闭11主机上的httpd,重新访问站点**
**查看12主机上的日志**
` tail -fn 10 /etc/httpd/logs/dz.abc.net_access_log`
![](_v_images/20191013124210966_5072.png)

### 再添加一台nginx主机(192.168.1.18)
`yum install nginx -y`
`cd /etc/nginx/`
`scp -r domains/ nginx.conf root@192.168.1.18:/etc/nginx/` #从而16主机复制到18主机
`nginx -s reload`
**如果16主机挂了，把电脑hosts文件改成192.168.1.18  dz.abc.net  wp.abc.net,确保服务在线，当然也可以用脚本实现主机的漂移。**







