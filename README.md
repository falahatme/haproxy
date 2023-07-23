```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http


listen Stats-Page
        bind *:8000
        mode http
        stats enable
        stats hide-version
        stats refresh 10s
        stats uri /
        stats show-legends
        stats show-node

frontend fe-apiserver
        bind 0.0.0.0:6443
        mode tcp
        option tcplog
        default_backend be-apiserver

backend be-apiserver
        mode tcp
        option tcp-check
        balance roundrobin
        default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
        server control-plane-1 192.168.15.133:6443 check
        server control-plane-2 192.168.15.134:6443 check
        server control-plane-3 192.168.15.135:6443 check

frontend http_frontend
        mode http
        bind *:80
        bind *:443 ssl crt /etc/ssl/certs/mysite.pem alpn h2,http/1.1  ssl-min-ver TLSv1.2
        redirect scheme https code 301 if !{ ssl_fc }
        default_backend   http_servers

backend http_servers
        mode http
        balance roundrobin
        option httpchk HEAD /
        http-response set-header X-Frame-Options SAMEORIGIN
        http-response set-header X-XSS-Protection 1;mode=block
        http-response set-header X-Content-Type-Options nosniff
        default-server check maxconn 5000
        server http_server1 192.168.15.133:80
        server http_server2 192.168.15.134:80
        server http_server3 192.168.15.135:80
```

## mysite.pem is :

cat mysite.key mysite.crt mysite.ca-bundle > mysite.pem

