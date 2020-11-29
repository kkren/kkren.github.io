---
title: 自建 Tiny Tiny RSS 踩坑记录
date: 2020-11-07 17:24:44
tags:
urlname: selfhost-ttrss
---
在此之前，我一直使用 telegram 上的 [@RustRssBot](https://t.me/RustRssBot) 来订阅 rss。使用telegram进行推送确实很爽，但是许多网站用此bot订阅时都会出现网络错误，而且我似乎也需要一款更专业的软件来阅读订阅。经过一番比较，我选择部署Tiny Tiny RSS。
<!-- more -->

## 部署

我使用的服务器是 AWS 位于俄勒冈的 lightsail 服务器，1核1g，Debian 10系统。部署 TTRSS 可以直接用 [Awesome-TTRSS](https://github.com/HenryQW/Awesome-TTRSS) 这个项目提供的 docker-compose 文件。首先安装好docker和docker-compose。

*[TTRSS]: Tiny Tiny RSS

*[AWS]: Amazon web services

```bash
wget https://github.com/HenryQW/Awesome-TTRSS/raw/master/docker-compose.yml
sudo docker-compose up -d
```

需要注意的是这个docker-compose.yml文件目前有一点小问题。用于处理全文输出的mercury容器是无法连上外部网络的。因为在下面的networks中，service-only这个network被设置为internal[^1]。将`internal: true` 删除即可。

[^1]: 即使不设置internal，容器也默认是不能从外部访问的，需要手动设置端口转发。

另外，TTRSS容器内部有一个文件夹用于存放站点的 favicon，这个文件夹并没有被持久化，所以如果重新创建容器，就会丢失这些favicon。需要等待12个小时才会重新刷新。我们可以创建一个volume来持久化这些favicon。修改后的docker-compose.yml如下（部分service省去）：

```yaml
version: "3"
services:
  service.rss:
    image: wangqiru/ttrss:latest
    container_name: ttrss
    ports:
      - 181:80
    environment:
      - SELF_URL_PATH=http://localhost:181/ # please change to your own domain
      - DB_PASS=ttrss # use the same password defined in `database.postgres`
    volumes:
      - feed-icons:/var/www/feed-icons/
    networks:
      - public_access
      - service_only
      - database_only
    stdin_open: true
    tty: true
    restart: always

  service.mercury: # set Mercury Parser API endpoint to `service.mercury:3000` on TTRSS plugin setting page
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    networks:
      - service_only
    restart: always

  database.postgres:
    image: postgres:13-alpine
    container_name: postgres
    environment:
      - POSTGRES_PASSWORD=ttrss # feel free to change the password
    volumes:
      - ~/postgres/data/:/var/lib/postgresql/data # persist postgres data to ~/postgres/data/ on the host
    networks:
      - database_only
    restart: always

networks:
  public_access: # Provide the access for ttrss UI
  service_only: # Provide the communication network between services only
  database_only: # Provide the communication between ttrss and database only
    internal: true
volumes:
  feed-icons:
```

再次执行`sudo docker-compose up -d`来更新容器。部署完容器后还需要安装nginx来反代服务。nginx的配置文件可以参考文档：[配置 HTTPS](https://ttrss.henry.wang/zh/#%E9%85%8D%E7%BD%AE-https) 。

### 部署 RSSHub 

 是一个开源、简单易用、易于扩展的 RSS 生成器，可以给任何奇奇怪怪的内容生成 RSS 订阅源。虽然 RSSHub 有官方提供的服务，但是几乎无法爬取任何反爬严格的站点。自建 RSSHub 依然可以使用 docker-compose。参考官方的[部署文档](https://docs.rsshub.app/install/#docker-compose-bu-shu)，可以快速启动一个RSSHub容器。注意，互联网[^2]上可能有人爬取自建的RSSHub。推荐不将RSSHub暴露到公网，或者使用http basic auth来保护。TTRSS支持在订阅时添加http basic auth的用户名与密码。

[^2]: https://www.v2ex.com/t/639512	 "V2EX - 用 RSS 订阅全平台的经历和体验"

### 插件

在TTRSS中，每个用户都可以独立配置插件，但是在docker的环境变量中可以全局开启特定插件

#### mercury

mercury是一个用于全文输出订阅源的插件，使用下来效果还是很准确的。之前我们启动的容器中已经包含了mercury的容器。接下来只需要在`偏好设置 - mercury_fulltext`启用插件即可。然后在`订阅源 - mercury fulltext settings`中，填写api地址为`service.mercury:3000`。还可以设置特定订阅源自动在收取时转换全文，在`编辑订阅源-插件`中启用即可。

#### fever

许多第三方客户端都需要 fever api 的支持。在`偏好设置 - Fever Emulation`中设置密码即可启用fever。此密码可以与账户密码相同也可以不同。注意使用fever api不能编辑、添加订阅源，也不支持分类嵌套等功能。 

## 订阅源

我订阅的大部分订阅源都是来自于个人博客，基本上都直接提供订阅源并且全文输出。但是有部分国内的社区不提供订阅源，或者需要进一步自定义订阅源。除了使用 RSSHub 以外，另一个工具是 [feed43](https://feed43.com/)。通过这个网站提供的查找与替换工具，可以很轻松制作订阅源。我给触乐网的[触乐夜话](http://www.chuapp.com/tag/index/id/20369.html)栏目制作了一个[订阅源](https://feed43.com/4458882322274404.xml)。可惜feed43没法很方便的全文输出，但是用TTRSS的mercury插件全文输出的效果很不错。

## 客户端

### ios&macos

| 名称                                 | api   | 授权 |
| ------------------------------------ | ----- | ---- |
| [Reeeder](https://www.reederapp.com/) | Fever | 收费 |

我没苹果设备，但是Reeder看起来很好看。

<img src="https://i.loli.net/2020/11/29/Ru5PE2DBkCwz6tG.png" alt="Reeder-screenshot" style="zoom:50%;" />

### Android

| 名称                                                         | api         | 授权      |
| ------------------------------------------------------------ | ----------- | --------- |
| [Feedme](https://play.google.com/store/apps/details?id=com.seazon.feedme) | Fever/TTRSS | 开源      |
| [Readably](https://play.google.com/store/apps/details?id=com.isaiasmatewos.readably) | Fever       | 免费/内购 |

Readably看起来很好看，但是Feedme也不错而且更接近Material Design。我选择了更简洁的Readably。

<img src="https://i.loli.net/2020/11/07/L9k5qO4uXletGgf.png" alt="Readably-screenshot" style="zoom:50%;" />

### Windows

| 名称                                                         | api   | 授权      |
| ------------------------------------------------------------ | ----- | --------- |
| [Fluent Reader](https://github.com/yang991178/fluent-reader) | Fever | 开源/收费 |

Fluent Reader是一个很新的rss客户端。在 Microsoft Store上是收费的，但是在Github release 上有免费的安装包。Fluent Reader使用electron+flutter的技术，外观还是很不错的。阅读体验也很好。当然，除了客户端，在Windows上直接使用网页端的体验也很好。

<img src="https://i.loli.net/2020/11/07/XIwMKuGlr7ytWhY.png" alt="Fluent Reader screenshot" style="zoom:50%;" />

## 结语

在2020年使用rss服务已经确实变得困难重重，rss的概念也与移动互联网的概念有了很大的差别。但是我还是习惯于使用rss收取信息。另一方面，我希望支持还在写个人博客的作者。我的订阅源中主要都是个人博客，其中的很多作者后来都成为了我的朋友，或者从我的朋友那里了解到了他们的个人博客。非常感谢这些博客在我最早使用互联网的时候给我带来的帮助。