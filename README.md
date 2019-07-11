# mediawiki
Bastille Template to create a MediaWiki Jail

Configure PHP

We'll configure PHP so that it uses a unix domain socket instead of TCP/IP. Edit /usr/local/etc/php-fpm.conf and change the listen directive:

	listen = /var/run/php-fpm.sock
	listen.owner = www
	listen.group = www
	listen.mode = 0660

Start PHP with

	sudo service php-fpm start

Test nginx & TLS

Use this test nginx config:

	worker_processes 1;
	events {
		worker_connections  1024;
	}

	http {
		server {
			listen 80;
			listen [::]:80;
			server_name xw.is;
			return 301 https://$server_name$request_uri;
		}
		server {
			listen 443;
			listen [::]:443;
			server_name xw.is;

			ssl on;
			ssl_certificate /tank/nginx/tls/xw.is.crt;
			ssl_certificate_key /tank/nginx/tls/xw.is.key;
			ssl_session_cache shared:SSL:1m;
			ssl_session_timeout 5m;

			ssl_ciphers HIGH:!aNULL:!MD5;
			ssl_prefer_server_ciphers on;

			location / {
				root /usr/local/www/nginx;
				index index.html index.htm;
			}
		}
	}


Start nginx:

	sudo service nginx start

Make sure everything works.


Enable the wiki

One everything works, add this block to nginx.conf.

	location /w {
		root /tank/www;
		index index.php;
		location ~ \.php$ {
			try_files $uri =404;
			fastcgi_split_path_info ^(.+\.php)(/.+)$;
			fastcgi_pass unix:/var/run/php-fpm.sock;
			fastcgi_index index.php;
			fastcgi_param SCRIPT_FILENAME $request_filename;
			include fastcgi_params;
		}
	}

and create this symlink:

	ln -s /usr/local/www/mediawiki /tank/www/w

Now visit visit https://xw.is/w and complete the installer.  The installer will generate 
a LocalSettings.php file. Copy it to the server:

	scp LocalSettings.php xw.is:/usr/local/www/mediawiki

Enable short URLs

To enable short URLs, use this nginx.conf config:

	worker_processes  1;
	events {
		worker_connections 1024;
	}

	http {
		include mime.types;
		default_type application/octet-stream;

		sendfile on;
		keepalive_timeout 65;

		server {
			listen 80;
			listen [::]:80;
			server_name xw.is;
			return 301 https://$server_name$request_uri;
		}

		server {
			listen 443;
			listen [::]:443;
			server_name xw.is;

			ssl on;
			ssl_certificate /tank/nginx/tls/xw.is.crt;
			ssl_certificate_key /tank/nginx/tls/xw.is.key;
			ssl_session_cache shared:SSL:1m;
			ssl_session_timeout 5m;

			ssl_ciphers HIGH:!aNULL:!MD5;
			ssl_prefer_server_ciphers on;

			root /tank/www;
			index index.php;

			location / {
				rewrite ^/$ https://xw.is/wiki permanent;
			}

			location /w {
				location ~ \.php$ {
					try_files $uri =404;
					fastcgi_split_path_info ^(.+\.php)(/.+)$;
					fastcgi_pass unix:/var/run/php-fpm.sock;
					fastcgi_index index.php;
					fastcgi_param SCRIPT_FILENAME $request_filename;
					include fastcgi_params;
				}
			}

			location /w/images {
				location ~ ^/w/images/thumb/(archive/)?[0-9a-f]/[0-9a-f][0-9a-f]/([^/]+)/([0-9]+)px-.*$ {
					try_files $uri $uri/ @thumb;
				}
			}
			location /w/images/deleted {
				# Deny access to deleted images folder
				deny all;
			}

			location /w/cache       { deny all; }
			location /w/languages   { deny all; }
			location /w/maintenance { deny all; }
			location /w/serialized  { deny all; }
			location ~ /.(svn|git)(/|$) { deny all; }
			location ~ /.ht { deny all; }

			location /wiki {
				include fastcgi_params;
				fastcgi_param SCRIPT_FILENAME $document_root/w/index.php;
				fastcgi_pass unix:/var/run/php-fpm.sock;
			}

			location @thumb {
				rewrite ^/w/images/thumb/[0-9a-f]/[0-9a-f][0-9a-f]/([^/]+)/([0-9]+)px-.*$ /w/thumb.php?f=$1&width=$2;
				rewrite ^/w/images/thumb/archive/[0-9a-f]/[0-9a-f][0-9a-f]/([^/]+)/([0-9]+)px-.*$ /w/thumb.php?f=$1&width=$2&archived=1;
				include fastcgi_params;
				fastcgi_param SCRIPT_FILENAME $document_root/w/thumb.php;
				fastcgi_pass unix:/var/run/php-fpm.sock;
			}

			error_page 500 502 503 504 /50x.html;
			location = /50x.html {
				root /usr/local/www/nginx-dist;
			}
		}
	}

Then edit LocalSettings.php to enable short URLs:

	$wgScriptPath = "/w";
	$wgScriptExtension = ".php";
	$wgArticlePath = "/wiki/$1";
	$wgUsePathInfo = true;

You are now done.


Enable mobile support

Make sure you have wget installed:

	sudo pkg install -y wget

Make sure you have permissions to add mediawiki extensions:

	sudo chown root:staff /usr/local/www/mediawiki/extensions
	sudo chmod g+w /usr/local/www/mediawiki/extensions

Download MobileFrontend extension and extract it:

	wget https://extdist.wmflabs.org/dist/extensions/MobileFrontend-REL1_27-717861c.tar.gz
	tar -xzf MobileFrontend-REL1_27-717861c.tar.gz -C /usr/local/www/mediawiki/extensions

Edit LocalSettings.php to enable it (append it at the bottom):

	wfLoadExtension( 'MobileFrontend' );
	$wgMFAutodetectMobileView = true;

