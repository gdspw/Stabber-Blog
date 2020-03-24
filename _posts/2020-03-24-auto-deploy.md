---
layout: post
title:  "使用Github的Webhooks+Node完成网站的自动化部署"
categories: it,github,node
tags:  [github,node,webhooks]
excerpt: 自打建立这个博客以来，从最开始的手动部署（纯人工），到通过服务器定时任务自动拉去最新代码部署（不及时），通过jenkins构建，到现在的通过github的webhooks自动化部署，并做此文章以记录...

---

前面也说到了，目前本博客网站的部署方式，是通过github的webhooks监听push事件来触发远程服务器的自动化部署脚本，如下：

本地环境->push到github仓库->触发远程自动化部署脚本->网站更新为最新的代码

## 1.配置github代码仓库的webhook

![9gzCC7](http://image.itstabber.com/uPic/9gzCC7.png)

如上图进入到项目仓库的settings下面，选中Webhooks选项进行配置即可:

>Payload Url：服务器地址+端口+自定义接口地址
>
>Content type: application/json
>
>Secret: 自定义
>
>**Which events would you like to trigger this webhook?** 
>
>勾选 Just the push event
>
>勾选 Active

配置好个选项后，点击**Update webhook**按钮即可。

## 2.服务器新增监听push事件的js脚本

### 定义服务脚本：deploy.js

**deploy.js**源码如下：

```javascript
var http = require('http')
var createHandler = require('github-webhook-handler')
var handler = createHandler({ path: '/webhook', secret: '自定义' })

function RunCmd(cmd, args, cb) {
  var spawn = require('child_process').spawn;
  var child = spawn(cmd, args);
  var result = '';
  child.stdout.on('data', function(data) {
    result += data.toString();
  });
  child.stdout.on('end', function() {
    cb(result)
  });
}

http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404;
    res.end('no such location');
  })
}).listen(7777)

handler.on('error', function (err) {
  console.error('Error:', err.message);
})

handler.on('push', function (event) {
  console.log('Received a push event for %s to %s',
    event.payload.repository.name,
    event.payload.ref);
  var shpath = './blog-start.sh';
  RunCmd('sh', [shpath], function(result) {
      console.log(result);
  })
})

handler.on('issues', function (event) {
  console.log('Received an issue event for %s action=%s: #%d %s',
    event.payload.repository.name,
    event.payload.action,
    event.payload.issue.number,
    event.payload.issue.title);
})
```

js脚本使用到了***github-webhook-handler***的一个组件，可以通过：  **npm install -g github-webhook-handler**来全局安装该组件，安装完组件后，直接启动该脚本时可能会遇到找不到该组件的错误，此时进入**deploy.js**脚本所在的目录下面执行：**npm link github-webhook-handler**

脚本中的**secret**需要跟刚才在github的webhook中**secret**的值一致。**blog-start.sh**是我博客的的启动脚本，稍后会展示源码。

脚本中的这段代码：

```java
http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404;
    res.end('no such location');
  })
}).listen(7777)
```

是创建服务，并监听7777端口，这个端口可以根据个人情况自行调整。

### 2.2 定义启动脚本

下面是**blog-start.sh**脚本的源码，仅供参考：

```shell
## 进入博客目录
echo '进入博客目录'
cd /myblog/Stabber-Blog
## 拉取最新代码
echo '开始拉去最新代码'
git pull
## 打包静态文件
echo '开始构建静态文件'
bundle exec jekyll build
echo '最新文章处理完成'
```

进行完以上步骤后，可以启动**deploy.js**脚本，执行以下指令，后台启动脚本：**nohup node deploy.js > deploy.log &**

### 2.3 配置nginx，做请求转发

因为为了服务器的安全性，服务器的7777端口我并未对外开放，此时需要配置服务器的nginx服务来转发对应的请求，配置参考如下：

```nginx
server {
        listen       80;
        server_name  www.itstabber.com;
  ...
  			location /webhook {
                proxy_pass http://localhost:7777/webhook;
        }
}
```

重启nginx是的配置生效： **./nginx -s reload**

## 3. 验证配置结果

到此，当对应的代码仓库有新的push内容后，就回自动触发node启动的js脚本服务，并且通过**blog-start.sh**自动的从github拉去最新的代码并完成代码构建，此时我的博客内容就会加入最新新增的文章信息了。下面截图是我本片文章新增时的验证结果（截图内容都是提交后补充的）：

### 提交本文章内容

![DUPHLj](http://image.itstabber.com/uPic/DUPHLj.png)

### 查看webhook调用结果

进入github项目仓库的webhooks目录下，可以看到本次push触发了接口调用，接口返回：

![GmV3AV](http://image.itstabber.com/uPic/GmV3AV.png)

### 查看服务器监听脚本日志

![image-20200325003634556](http://image.itstabber.com/uPic/image-20200325003634556.png)

日志显示已经自动拉取刚刚提交的文章，并且重新构建完成

### 访问博客网站，进行结果验证

![image-20200325005159802](http://image.itstabber.com/uPic/image-20200325005159802.png)

截图中可以看到，文章已经发布到网站上了。

看完文章后尝试自己配置的小伙伴们，如果又遇到什么问题，可以在文章下方评论区留言交流，多谢！