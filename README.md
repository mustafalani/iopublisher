# iopublisher
iohub Stream Publisher 

Download the latest nginx source

		wget http://nginx.org/download/nginx-x.x.x.tar.gz
		tar -xf nginx-x.x.x.tar.gz

Download nginx RTMP module	
	
		git clone https://github.com/sergey-dryabzhinsky/nginx-rtmp-module.git

Install nginx
	
		cd nginx-x.x.x
		./configure --with-http_ssl_module --add-module=../nginx-rtmp-module --with-debug
		make -j 1
		sudo make install

Add nginx log directory
		
		sudo mkdir /var/log/nginx/

Add the following to nginx configuration or replace the configuration file with nginx.conf
	
		nano /usr/local/nginx/conf/nginx.conf

			user                        www-data;
			#pid                         /run/nginx.pid;
			worker_processes            8;

			#error_log  logs/error.log;
			#error_log  logs/error.log  notice;
			#error_log  logs/error.log  info;

			#pid        logs/nginx.pid;


			events {
			    worker_connections  1024;
			}


			http {
			    include       mime.types;
			    default_type  application/octet-stream;

			    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
			    #                  '$status $body_bytes_sent "$http_referer" '
			    #                  '"$http_user_agent" "$http_x_forwarded_for"';

			    #access_log  logs/access.log  main;

			    sendfile        on;
			    #tcp_nopush     on;

			    #keepalive_timeout  0;
			    keepalive_timeout  65;

			    #gzip  on;
			    
			    server {
			    listen       8080;
			    include /usr/local/iohub/publisher/applications/*/http.conf;
			    # Logging Settings

			    access_log              /var/log/nginx/access.log;
			    error_log               /var/log/nginx/error.log;
			    }


			    server {
			        listen       80;
					server_name  localhost;

			        #charset koi8-r;

			        #access_log  logs/host.access.log  main;

			        location / {
			            root   html;
			            index  index.html index.htm;
			        }
			        #include /usr/local/iohub/publisher/applications/*/http.conf;

			        #error_page  404              /404.html;

			        # redirect server error pages to the static page /50x.html
			        #
			        error_page   500 502 503 504  /50x.html;
			        location = /50x.html {
			            root   html;
			        }

			        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
			        #
			        #location ~ \.php$ {
			        #    proxy_pass   http://127.0.0.1;
			        #}

			        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
			        #
			        #location ~ \.php$ {
			        #    root           html;
			        #    fastcgi_pass   127.0.0.1:9000;
			        #    fastcgi_index  index.php;
			        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
			        #    include        fastcgi_params;
			        #}

			        # deny access to .htaccess files, if Apache's document root
			        # concurs with nginx's one
			        #
			        #location ~ /\.ht {
			        #    deny  all;
			        #}
			    }    
				# another virtual host using mix of IP-, name-, and port-based configuration
			    #
			    #server {
			    #    listen       8000;
			    #    listen       somename:8080;
			    #    server_name  somename  alias  another.alias;

			    #    location / {
			    #        root   html;
			    #        index  index.html index.htm;
			    #    }
			    #}


			    # HTTPS server
			    #
			    #server {
			    #    listen       443 ssl;
			    #    server_name  localhost;

			    #    ssl_certificate      cert.pem;
			    #    ssl_certificate_key  cert.key;

			    #    ssl_session_cache    shared:SSL:1m;
			    #    ssl_session_timeout  5m;

			    #    ssl_ciphers  HIGH:!aNULL:!MD5;
			    #    ssl_prefer_server_ciphers  on;

			    #    location / {
			    #        root   html;
			    #        index  index.html index.htm;
			    #    }
			    #}

			}
			#include /usr/local/iohub/publisher/applications/*/Application.conf;
			rtmp {
			    server {
			        listen 1935; # Listen on standard RTMP port
			        chunk_size 4000;
			        include /usr/local/iohub/publisher/applications/*/Application.conf;
			    }
			}

Add nginx service
		
		sudo nano /lib/systemd/system/nginx.service

		[Unit]
		Description=The NGINX HTTP and reverse proxy server
		After=syslog.target network.target remote-fs.target nss-lookup.target

		[Service]
		Type=forking
		PIDFile=/run/nginx.pid
		ExecStartPre=/usr/local/nginx/sbin/nginx -t
		ExecStart=/usr/local/nginx/sbin/nginx
		ExecReload=/bin/kill -s HUP $MAINPID
		ExecStop=/bin/kill -s QUIT $MAINPID
		PrivateTmp=true

		[Install]
		WantedBy=multi-user.target


Add the following directories:

		mkdir /usr/local/iohub/publisher/applications
		mkdir /mnt/hls/

Chnage ownership of hls directory

		chown -R www-data:www-data hls

make a directory for each application inside
		
		mkdir /usr/local/iohub/publisher/applications/$app_name

Add two new configuration files inside each application

		nano /usr/local/iohub/publisher/applications/$app_name/Application.conf

		# RTMP configuration
        application $app_name {
            live on;
            # Turn on HLS
            hls on;
            hls_path /mnt/hls/$app_name;
            hls_fragment 3;
            hls_playlist_length 60;
            # disable consuming the stream from nginx as rtmp
            allow publish all;
            allow play all;
        }

		nano /usr/local/iohub/publisher/applications/$app_name/http.conf

		location /$app_name {
        # Disable cache
        add_header 'Cache-Control' 'no-cache';
        # CORS setup
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Expose-Headers' 'Content-Length';

        # allow CORS preflight requests
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        types {
            application/dash+xml mpd;
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
        }

        root /mnt/hls/;
    	}

Test nginx configuration
		
		/usr/local/nginx/sbin/nginx -t

Start nginx in the background:
		
		sudo /usr/local/nginx/sbin/nginx

Stop nginx:
		
		sudo /usr/local/nginx/sbin/nginx -s stop
