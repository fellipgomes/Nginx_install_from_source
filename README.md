# Nginx_install_from_source

This is a how-to guide to install NGINX from Source Code
Hope that will help someone :-)
I plan to write some tutorials very soon on subjects such as: Kubernetes, Elasticsearch, Terraform, etc
You can follow me on twitter: @realMilord
------------------------------------------------

Update the Package List
$ sudo apt update 

$ sudo apt upgrade -y

$ mkdir download && cd download

$ wget https://nginx.org/download/nginx-1.18.0.tar.gz

Unpacked file 
$ tar -zxvf nginx-1.18.0.tar.gz

Navigate to the newly unpacked folder
$ cd  nginx-1.18.0

To list the files in the directory
$ ls -la

Try to configure now (to see the missing libraries needed to compile the source code)
$  ./configure 

Install the C compiler 
$ sudo apt install build-essential  -y

Re-run the configure to check the missing libraries needed for compiling the source code
$  ./configure 

Install the necessary dev tools for compilation 
$ sudo apt install libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev -y

Configure with custom flags
For security reason: remove the following module --without-http_autoindex_module 
see: http://nginx.org/en/docs/configure.html 

If you plan to add NGINX in front DBs like MongoDB, add the following flag while compiling:
(I will write a tutorial on that later, how to put NGINX as a proxy in front of a MONGODB Cluster with https)
--with-stream for MONGODB

$ ./configure --sbin-path=/usr/bin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/acccess.log --with-pcre --pid-path=/var/run/nginx.pid --with-http_ssl_module  --with-http_v2_module --without-http_autoindex_module

Make the configuration
$ make 

Install 
$ sudo make install 

Now test NGINX is working 
$ nginx -V

Get the IP address of the instance to check NGINX in the browser
Get the inet of the 
$ ifconfig 

Now to use the nginx service with the systemd, to be able to do things like sudo systemctl start nginx,
you will need to create the service for the systemd.

Create the service for system
1 - create the file /lib/systemd/system/nginx.service
$ sudo vi /lib/systemd/system/nginx.service

2-  Copy the following: 
 (for more info, see example: https://www.nginx.com/resources/wiki/start/topics/examples/systemd/ )

Add the following in the file that you neewly created (nginx.service)

[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/bin/nginx -t
ExecStart=/usr/bin/nginx
ExecReload=/usr/bin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target


Now we can use systemctl to interact wit nginx

Reload the systemd deamon 
$ sudo systemctl daemon-reload 

$ sudo systemctl status nginx

Start the service 
$ sudo systemctl start nginx

To allow the service to start after reboot
$ sudo systemctl enable nginx 


----------
if encounter the following error:
nginx.service: Failed to parse PID from file /var/run/nginx.pid: Invalid argument

Do the following:
https://bugs.launchpad.net/ubuntu/+source/nginx/+bug/1581864 

Here the workaround:
[Workaround]
sudo mkdir /etc/systemd/system/nginx.service.d
printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" | \
sudo tee /etc/systemd/system/nginx.service.d/override.conf
sudo systemctl daemon-reload
sudo systemctl restart nginx


# Outro metodo de instalar

https://serverfault.com/questions/879950/how-to-resolve-openssl-package-error-while-installing-nginx

sudo yum install unzip gcc pcre-devel zlib-devel make golang wget
mkdir -p /tmp/nginx-dep
cd /tmp/nginx-dep
curl -O https://www.openssl.org/source/openssl-1.0.2g.tar.gz
curl -O http://nginx.org/download/nginx-1.9.14.tar.gz
tar zxf openssl-1.0.2g.tar.gz
tar zxf nginx-1.9.14.tar.gz
cd nginx-1.9.14/
./configure --with-http_ssl_module \
            --with-openssl=`realpath ../openssl-1.0.2g` \
            --prefix=/usr/share/nginx \
            --sbin-path=/usr/sbin/nginx \
            --modules-path=/usr/lib/nginx/modules \
            --conf-path=/etc/nginx/nginx.conf \
            --error-log-path=/var/log/nginx/error.log \
            --http-log-path=/var/log/nginx/access.log \
            --pid-path=/run/nginx.pid \
            --lock-path=/var/lock/nginx.lock \
            --http-client-body-temp-path=/var/lib/nginx/body \
            --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
            --http-proxy-temp-path=/var/lib/nginx/proxy \
            --http-scgi-temp-path=/var/lib/nginx/scgi \
            --http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
            --with-openssl-opt=enable-ec_nistp_64_gcc_128 \
            --with-openssl-opt=no-nextprotoneg \
            --with-openssl-opt=no-weak-ssl-ciphers \
            --with-openssl-opt=no-ssl3 \
            --with-pcre-jit \
            --with-threads \
            --with-http_addition_module \
            --with-http_auth_request_module \
            --with-http_dav_module \
            --with-http_flv_module \
            --with-http_gunzip_module \
            --with-http_gzip_static_module \
            --with-http_mp4_module \
            --with-http_random_index_module \
            --with-http_realip_module \
            --with-http_slice_module \
            --with-http_ssl_module \
            --with-http_sub_module \
            --with-http_stub_status_module \
            --with-http_v2_module \
            --with-http_secure_link_module \
            --with-mail \
            --with-mail_ssl_module \
            --with-stream \
            --with-stream_ssl_module
make
sudo make install

sudo mkdir -p /var/lib/nginx && sudo nginx -t

# create service
sudo touch /etc/systemd/system/nginx.service
printf '
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
 Nginx will fail to start if /run/nginx.pid already exists but has the wrong
 SELinux context. This might happen when running `nginx -t` from the cmdline.
 https://bugzilla.redhat.com/show_bug.cgi?id=1268621
ExecStartPre=/usr/bin/rm -f /run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
KillSignal=SIGQUIT
TimeoutStopSec=5
KillMode=process
PrivateTmp=true

[Install]
WantedBy=multi-user.target' | sudo tee  /etc/systemd/system/nginx.service

rm /tmp/nginx-dep -ri
