This document is for those who use laravel echo server & nginx & socket.io & redis-server with separated server between client project and redis-server.

**1) Edit `/etc/redis/redis.conf`**

```php
bind 127.0.0.1
supervised no
```

To

```php
bind 0.0.0.0
supervised systemd
```

**2) Update `/etc/systemd/system/redis.service` under `[Service]`**

```php
Type=notify
ExecStart=/usr/bin/redis-server /etc/redis/redis.conf  --supervised systemd
```

**3) Nginx `/etc/nginx/sites-enabled/reverse-proxy.conf`**

```php
server {
  listen        443 ssl;
  listen        [::]:443 ssl;
  server_name   mysitecom;

  error_log     /var/log/nginx/proxy-error.log error;

  # Start the SSL configurations
  ssl                         on;
  ssl_certificate             /etc/nginx/certs/mysitecom.pem;
  ssl_certificate_key         /etc/nginx/certs/mysitecom.key;
  ssl_session_timeout         3m;
  ssl_session_cache           shared:SSL:50m;
  ssl_protocols               TLSv1.1 TLSv1.2;

  # Diffie Hellmann performance improvements
  ssl_ecdh_curve              secp384r1;

  location /socket.io {
    proxy_pass                          http://mysitecom:2096;
    proxy_http_version 1.1;
    proxy_set_header Upgrade            $http_upgrade;
    proxy_set_header Connection         "upgrade";
    proxy_set_header Host               $host;
    proxy_set_header X-Real-IP          $remote_addr;
    proxy_buffers 16 4k;
    proxy_buffer_size 2k;

    proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto  https;
    proxy_set_header X-VerifiedViaNginx yes;
    proxy_read_timeout                  2h;
    proxy_connect_timeout               2h;
    proxy_redirect                      off;
  }
}
```

**4) laravel-echo-server.json**

```php
{
	"authHost": "https://mysitecom",
	"authEndpoint": "/broadcasting/auth",
	"clients": [
		{
			"appId": "e45c056ec8ca8bd7",
			"key": "88d316b5cccafbc5e905aa9ee13e63f7"
		}
	],
	"database": "redis",
	"databaseConfig": {
		"redis": {
			"host": "0.0.0.0",
			"port": "6379"
		},
		"sqlite": {
			"databasePath": "/database/laravel-echo-server.sqlite"
		}
	},
	"devMode": true,
	"host": null,
	"port": "2096",
	"protocol": "https",
	"socketio": {},
	"secureOptions": 67108864,
	"sslCertPath": "/etc/nginx/certs/mysitecom.pem",
	"sslKeyPath": "/etc/nginx/certs/mysitecom.key",
	"sslCertChainPath": "",
	"sslPassphrase": "",
	"subscribers": {
		"http": true,
		"redis": true
	},
	"apiOriginAllow": {
		"allowCors": true,
		"allowOrigin": "*",
		"allowMethods": "GET, POST",
		"allowHeaders": "Origin, Content-Type, X-Auth-Token, X-Requested-With, Accept, Authorization, X-CSRF-TOKEN, X-Socket-Id"
	}
}
```

**Note:** for someone who connects DNS with `cloudflare` please change default socket.io port 6001 to the following [here][1].


  [1]: https://support.cloudflare.com/hc/en-us/articles/200169156-Which-ports-will-CloudFlare-work-with-
