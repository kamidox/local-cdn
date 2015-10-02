# 搭建本地 CDN 加速 StackOverflow 等网站的访问速度。

> **安全预警**：最近的 xcode ghost 事件让大家的安全意识提高了不少。本文介绍的方法可能存在一些安全隐患。当然，解决方案也是有的，就是可以把本地 cdn 加速的文件和官网发布的正式版本做 md5 检验，这样确保本地文件没有被修改。由于目前还没有实现 md5 检验。请确保你自己清楚你在做什么，否则可能导致你的计算机被恶意的人植入后门。

## 问题

访问 StackOverflow 时奇慢无比有没有？其实原因不是因为 StackOverflow 不能访问，而是因为 StackOverflow 网站引用了 Google 的 CDN 加速服务器来下载 jqeury 脚本。[知乎上有个话题][1]讨论了这个问题。

## 解决方案

在本地搭建一个 web 服务器，然后下载好 jquery 脚本，放在本地 web 服务器上。再个性 hosts 文件，把 ajax.googleapis.com 重定向到本地 127.0.0.1 。这样浏览器就会从本地下载脚本，而不再从 ajax.googleapis.com 下载了。

## 详细步骤

这里将介绍详细的安装和配置过程，需要注意的是所有的目录和命令以我的 mac 下的环境为例。如果你用的是不同的版本或不同的操作系统，根据你的情况完成相应的配置即可。

### 安装 nginx 服务器

网上一堆教程，搜索一下就可以成功安装，需要注意的是两点。一是需要设置 nginx 开机启动。二是需要修改 nginx 运行权限，因为我们要在 80 端口提供服务，80 端口需要 root 权限才可以。

**设置 nginx 开机启动**

把下面文件内容保存到 `/Library/LaunchAgents/com.nginx.plist`。然后运行 `launchctl load -w /Library/LaunchAgents/com.nginx.plist`。

```xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>homebrew.mxcl.nginx</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <false/>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/opt/nginx/bin/nginx</string>
        <string>-g</string>
        <string>daemon off;</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/usr/local</string>
    <key>StandardErrorPath</key>
    <string>/usr/local/var/log/nginx/error.log</string>
    <key>StandardOutPath</key>
    <string>/usr/local/var/log/nginx/access.log</string>
  </dict>
</plist>
```

**修改 nginx 运行权限**

```shell
sudo chown root:admin /usr/local/Cellar/nginx/1.8.0/bin/nginx
sudo chmod u+s /usr/local/Cellar/nginx/1.8.0/bin/nginx
```

### 配置 nginx 服务器

配置 nginx 服务器主要分两步。一是生成 ssl 证书，用来提供 https 服务。二是下载 local-cdn 到本地，将配置 nginx 指向这个本地目录以便提供 cdn 服务。

**生成 ssl 证书**

我们本地 cdn 还需要支持 ssl，所以需要做一个自签名的证书。下面命令会在 `/usr/local/etc/nginx/ssl/` 目录下生成一个自签名的证书。这个证书后面会用到。

```shell
mkdir -p /usr/local/etc/nginx/ssl
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /usr/local/etc/nginx/ssl/nginx.key -out /usr/local/etc/nginx/ssl/nginx.crt
```

**配置 nginx 服务器**

把 local-cdn 下载到本地，比如放在 `/Users/kamidox/work/local-cdn`。

```shell
cd /Users/kamidox/work/
git clone https://github.com/kamidox/local-cdn.git
```

然后配置 nginx 添加这个目录当作内容目录。我们把下面内容保存到 `/usr/local/etc/nginx/servers/local-cdn.conf`。注意配置中用到了我们上面生成的证书。另外，nginx 默认会 include `/usr/local/etc/nginx/servers/` 下的所有配置文件。如果你修改过 nginx 的默认配置，根据你自己的情况配置即可。

```yml
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    charset utf-8;

    listen 443 ssl;

    server_name localhost;
    ssl_certificate /usr/local/etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /usr/local/etc/nginx/ssl/nginx.key;

    location / {
        root /Users/kamidox/work/local-cdn/;
        index index.html index.htm;
    }
}
```

## Contribution

欢迎大家[提 pull request][2]，一起完善科学文明的上网环境。


[1]: http://www.zhihu.com/question/22909851
[2]: https://github.com/kamidox/local-cdn
