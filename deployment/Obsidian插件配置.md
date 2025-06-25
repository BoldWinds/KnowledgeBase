使用的插件尽可能的不要给Obsidian增加新的语法，维持Markdown的简洁性，只添加必要的功能性插件。
## 远程同步

使用插件[Remotely Save](https://github.com/remotely-save/remotely-save)。有两个同步方案：
1. 使用nextcloud自带的webdav同步
2. 使用阿里云（或其他云服务提供商）OSS方案同步
按理来说应该是方案2要更好，因为有免费额度，不用自己运维。但是谁让我已经搭建了Nextcloud。于是先用Nextcloud的自带webdav尝试一下。

注意一点，远程同步会默认在webdav服务的顶级目录下直接保存资料库的名字；而如果选择自定义目录，则不管哪个资料库都会放到那个目录下面。为解决这个问题，在webdav的url后面加上想要存储的目录，比如我想把所有资料库，每个资料库作为一个目录存储在Obsidian目录中，就用原始url加上'/Obsidian'即可。

## 图床

推荐最多的是使用[Image Auto Upload]()这个插件，但是这个插件竟然需要我启动一个PicGo作为图片上传服务。我就不明白了上传图片不就是一个请求的事情吗？但是当我安装Piclist[[图片上传服务]]
之后，他虽然不用我在每台机器上都安装PicGo，但是它不可以用时间戳自动改名😅，我投降了，我使用PicGo。

后续有时间可以修改一下PicList的源码，让他支持时间戳命名，打包个镜像自己用。

## 代理

网络问题一直让人非常头疼。Obsidian不支持直接设置代理，需要安装插件实现代理，使用[Global Proxy](https://github.com/windingblack/obsidian-global-proxy)。

## 自动补全

自动补全是一个非常重要的功能，自己写笔记的时候，很多句子其实Copilot都能比较好的生成出来，所以就不转门调别的API了，直接使用[Github Compilot](https://github.com/Pierrad/obsidian-github-copilot)。缺点就是这个插件依赖node。


