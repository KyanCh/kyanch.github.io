---
layout: post
title: 通过Nginx设置你的git服务器
date: 2024-10-24 05:57 +0800
---

> 参考[git http-protocl](https://git-scm.com/docs/http-protocol)

> 参考Blog [Setting up a Git HTTP server with Nginx](https://esc.sh/blog/setting-up-a-git-http-server-with-nginx/)

git已经带有一个git-http-backend程序提供http访问，这里只是用nginx反向代理这个服务。
另外我们还准备建立一个gitweb的服务，就像[linux kernel的git页面](https://git.kernel.org/)

```text
server {
	listen 80;
	listen [::]:80;

	server_name git.home.me;

	# For gitweb
	location / {
		if ($arg_service ~ "^git-(upload|receive)-pack$") {
		#if ($content_type ~ "application/x-git-(upload|receive)-pack-request") {
			rewrite ^/(.*) /git/$1?$args last;
		}
                root /usr/share/gitweb/;
                index index.cgi;
	}

	location /index.cgi {
                root /usr/share/gitweb/;
                include fastcgi_params;
                gzip off;
                fastcgi_pass unix:/var/run/fcgiwrap.socket;
                fastcgi_param SCRIPT_NAME $uri;
                fastcgi_param GITWEB_CONFIG /etc/gitweb.conf;
        }

	# Accelerated static for git smart http 
	location ~ "^/(.*/objects/[0-9a-f]{2}/[0-9a-f]{38})$" {
		root /srv/git;
	}
	location ~ "^/(.*/objects/pack/pack-[0-9a-f]{40}.(pack|idx))$" {
		root /srv/git;
	}
	# For git smart http
	location ~ ^/git(/.*) {
		root /srv/git;
		auth_basic "Restricted";
		auth_basic_user_file /etc/nginx/.gitpasswd;
		fastcgi_pass unix:/var/run/fcgiwrap.socket;
		include fastcgi_params;
		fastcgi_param SCRIPT_FILENAME /usr/lib/git-core/git-http-backend;
		fastcgi_param GIT_PROJECT_ROOT /srv/git;
		fastcgi_param GIT_HTTP_EXPORT_ALL "";
		fastcgi_param PATH_INFO		$1;
	}
	location ~ ^(/.*git-(upload|receive)-pack)$ {
		rewrite ^/(.*) /git/$1?$args last;
	}

}
```
