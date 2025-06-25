为了能够直接远程访问我的unraid控制界面，需要为其配置证书，unraid默认证书是自签名的，非常难用。

## 获取源证书

以下所有操作均根据cloudflare操作，如果是在其他地方注册的域名，可以将域名托管到cloudflare再进行操作。

点击你的域名，转到SSL/TLS，源服务器， 即可看到右方可以创建证书。该证书由Cloudflare签名，最长可以申请15年的域名。创建完毕后，复制证书到本地，并将cloudflare连接方式设置为“完全（严格）”。

## 替换证书

首先打开unraid去settings中的management access，将Use SSL/TLS设置为yes，并且设置Local TLD为要使用域名的根域名，比如要用a.b.com访问unraid，就设置为b.com，a则为你机器的名称。

> 这里注意，要提前设置好，域名=机器名.(Local TLD)，否则unraid会重新生成自签名证书。

在unraid中，没有区分.pem和.key，反而是将key拼接在pem的后面。找到unraid存放证书的位置`/boot/config/ssl/certs/BoldNas_unraid_bundle.pem`。接下来打开终端，将下载下来的证书粘贴替换unraid中保存的证书即可。

## 远程访问

一般来说，只要正确的把证书和key替换进去，unraid自己就会识别出签名证书的组织等信息。之后，只要开启unraid connect并且在unraid中登陆自己的账号，就可以在[[https://connect.myunraid.net/dashboard]]查看自己的unraid服务器状态。

但是，我还是用sakura frp设置了内网穿透。unraid配置sakura frp的方法见此[[配置内网穿透]]。方法很简单：
1. 在sakura frp中配置HTTPS类型的隧道，并设置域名，不要强制开启HTTPS
2. 复制节点的真实域名
3. 在cloudflare中，为域名设置`CNAME`类型的解析，指向sakura节点的域名
完成以上步骤后，应该就可以正常访问了！

## PS: cloudflare的证书

用cloudflare代理时，流量会先经过cloudflare的服务器，然后再由cloudflare与真实的网站服务器进行沟通，这期间需要用到两个证书：
- 边缘证书：用于客户端和cloudflare服务器的认证
- 源机器：用于cloudflare与真实服务器的认证