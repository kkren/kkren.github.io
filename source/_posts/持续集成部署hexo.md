---
title: 持续集成部署hexo
date: 2018-12-20 21:39:47
tags: 技术
urlname: hexo-with-travis
---

过去一直在写完博客后手动生成静态文件，然后丢到服务器上。但是在多终端上不方便保持统一的体验。解决这个问题的办法是使用 Travis ci 自动编译 hexo 的静态文件，然后通过 GitHub webhook 通知服务器更新网站。<!-- more -->

## 创建 GitHub repo

使用 `git clone` 将 repo 下载到本地后，执行 `git checkout hexo` 创建一个 hexo 分支，然后把原来的 hexo 文件夹全部复制进来。如果你之前曾用 git clone 的方式安装过主题，建议将主题目录里的 .git 文件删除，或者改用 submodule 的方式引用主题项目（不推荐）。

## 创建 Travis ci 配置

复制以下代码，**修改里面有关用户名和仓库地址的信息后**储存为 .travis.yml 在 repo 目录中。

```yaml
language: node_js
node_js: stable
branchs:
  only:
  - hexo
cache:
  directories:
  - node_modules
before_install:
# 安装hexo-cli
- npm install -g hexo-cli
install:
# 安装依赖包
- npm install
script:
- hexo g
after_script:
- git clone https://${GH_REF} .deploy_git
- cd .deploy_git
- git checkout master
- cd ../
- mv .deploy_git/.git/ ./public/
- cd ./public
- git config --global user.email "example@gmail.com"
- git config --global user.name "username" 
- git add .
- git commit -m "Travis CI Auto Builder"
- git push --force --quiet "https://${GH_TOKEN}@${GH_REF}" master:master
env:
    global:
        - GH_REF: github.com/username/username.github.io.git 
```

GH_TOKEN 需要加密后储存在配置文件中，打开 [Github设置]( https://github.com/settings/tokens/new) 创建一个新的 token，按下图配置权限即可：![token](https://i.loli.net/2018/12/20/5c1b939e53817.png)

然后，在安装好 ruby 的环境下，进入到 repo 的目录执行下面的命令，就能把 token 加密后添加到 Travis 配置中。

```bash
gem install travis
travis login
# 交互式完成登陆
travis encrypt 'GH_TOKEN=< 这里填入你生成的 Token >' --add
```

在 Linux 环境下，通过包管理器就能安装 ruby，具体步骤请查看[官方文档](https://www.ruby-lang.org/zh_cn/documentation/installation/)。

## 启用 Travis ci

在 [Travis ci 官网](https://travis-ci.org/)用自己的 GitHub 账户登陆，然后选择你网站的 repo，编译应该会自动开始。一切正常的话，编译结束后你 repo master 分支已经部署好了网站的静态文件。整个编译的日志都会被记录下来。常见问题的解决方法：

### 需要通过包管理器安装其他依赖

在`npm install`上添加两行：

```yaml
- sudo apt-get update
- sudo apt-get install -y package
```

`sudo` 和 `apt-get update` 都是必须的

## 接收 webhook

新建一个文件夹保存 webhook.js 和 deploy.sh，在这个文件夹下执行 `npm install github-webhook-handler`。将以下代码保存到 webhook.js：

```javascript
var http = require('http')
var createHandler = require('github-webhook-handler')
var handler = createHandler({
	path: '/deploy',
	secret: 'token'
})
// 上面的 secret 保持和 GitHub 后台设置的一致

function run_cmd(cmd, args, callback) {
	var spawn = require('child_process').spawn;
	var child = spawn(cmd, args);
	var resp = "";

	child.stdout.on('data', function(buffer) {
		resp += buffer.toString();
	});
	child.stdout.on('end', function() {
		callback(resp)
	});
}

http.createServer(function(req, res) {
	handler(req, res, function(err) {
		res.statusCode = 404
		res.end('no such location')
	})
}).listen(7777)

handler.on('error', function(err) {
	console.error('Error:', err.message)
})

handler.on('push', function(event) {
	console.log('Received a push event for %s to %s',
		event.payload.repository.name,
		event.payload.ref);
	run_cmd('sh', ['./deploy.sh', event.payload.repository.name], function(text) {
		console.log(text)
	});
})
```

将以下代码保存到 deploy.sh：

```bash
#!/bin/bash

WEB_PATH='/var/www/'$1

cd $WEB_PATH
echo "pulling source code at $WEB_PATH..."
git pull
```

假设网站目录在 /var/www/ 下，以 repo 名命名。然后执行 :

```bash
npm install pm2 -g
pm2 start webhook.js
```

 安装 pm2 来保持 webhook 运行。然后在 nginx 的站点配置文件中添加：

```nginx
location /deploy {
	proxy_pass http://127.0.0.1:7777;
}
```

重载 nginx，然后提前从 GitHub clone 下网站文件到相应的目录。

## 添加 webhook

打开 repo 的 settings-webhooks，点击 add webhook，按下图配置：
![webhook](https://i.loli.net/2018/12/20/5c1b9a248b403.png)
secret 要和 webhook.js 中的相同。添加以后，如果 request 正确的返回了 200，网站应该自动更新完毕了。