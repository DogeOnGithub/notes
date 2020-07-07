# Nginx 转发 websocket

## nginx.conf

在`http{}`中添加配置

```conf
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```

map指令的作用：

+ 该作用主要是根据客户端请求中 `$http_upgrade` 的值，来构造改变 `$connection_upgrade` 的值，即根据变量 `$http_upgrade` 的值创建新的变量 `connection_upgrade`
+ 创建的规则就是{}里面的东西。其中的规则没有做匹配，因此使用默认的，即 `$connection_upgrade` 的值会一直是 `upgrade`
+ 如果 $http_upgrade为空字符串的话，那值会是 close

添加`location`配置

```conf
location /xxx/xx/ {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $host;
    proxy_set_header X-Forworded-For $http_x_forworded_for;
    proxy_http_version 1.1;
    proxy_read_timeout 360s;
    proxy_redirect off;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header  X-Forwarded-Proto $scheme;
    proxy_pass http://xxx.xxx.xxx.xxx:xxxx/xxx/xxx/;
}
```

重点在于以下3个配置：

+ `proxy_http_version 1.1;`
+ `proxy_set_header Upgrade $http_upgrade;`
+ `proxy_set_header Connection "Upgrade";`

更详细请参考：[Nginx支持WebSocket反向代理-学习小结](https://www.cnblogs.com/kevingrace/p/9512287.html)
