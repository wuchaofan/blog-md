###搭建Sentry以及配合sourcemap使用

再次记录一下搭建sentry的流程及遇到的问题，作为演示采用的mackbook pro，这里使用docker环境搭建，首先下载[docker](<https://hub.docker.com/editions/community/docker-ce-desktop-mac>)。这里不介绍docker了，不太了解的话，网上教程很多，主要有docker、docker-compose工具链。

#####1、首先打开[onpremise](<https://github.com/getsentry/onpremise>) ，里面有基本的步骤，但是在此之前要注意clone这个项目后，里面的docker-compose.yml文件可能需要修改，我在搭建时遇到了一类问题`AssertionError: u'mail.port' cannot be changed at runtime because it is configured on disk,` ，在这个 [issue](https://github.com/getsentry/sentry/issues/12722#issuecomment-481949199) 里找到解决办法：

```yaml
auth.allow-registration: false
beacon.anonymous: true
mail.from: ""
mail.host: ""
mail.password: ""
mail.port: 465
mail.use-tls: true
mail.username: ""
system.admin-email: ""
system.url-prefix: ""
```

配置发邮件的信息，但是由于出错，直接把上面的配置拷贝到docker-compose.yml文件中。进入onpremise文件夹下面就是基本的步骤了：

```bash
1. docker volume create --name=sentry-data && docker volume create --name=sentry-postgres
2. cp -n .env.example .env
3. docker-compose build
4. docker-compose run --rm web config generate-secret-key
5. docker-compose run --rm web upgrade
6. docker-compose up -d
```

其中第3步产生的密钥，拷贝到第2步创建的.env文件里。如下图：

![image-20190420113832538](./img/image-20190420113832538.png)

第5步会询问是否创建管理员账户，可以先不创建，我在这一步时创建账户python报错了，我又重来了一次后选的是先不创建。这里在说下如果重新安装主要知道几个命令。

```bash
docker volume ls
```

输出如下：

![image-20190420114508569](./img/image-20190420114508569.png)

我们要删除这些数据的话需要利用`docker volume`

```bash
docker volume prune #全部删除
docker volume rm volume_name #制定名称
```

#####2、根据情况任选上面命令其一。当然如果一切顺利上面的重新安装步骤可以忽略，然后我们就可以在浏览器中打开sentry web网站了。

![image-20190420115104290](./img/image-20190420115104290.png)

上图为运行sentry依赖的几个容器，绿框可以开始web端口为9000，如下图：

![image-20190420115400614](./img/image-20190420115400614.png)

到了这一步需要账户，上面有提到，如果安装时你创建了账户登陆即可，如果没有创建，怎么创建账户呢，如下：

```bash
docker-compose run --rm web bash
```

这条命令会创建进入sentry web容器，进入容器后如下图：

![image-20190420115813682](./img/image-20190420115813682.png)

根据这些参数即可创建普通用户或者超级用户，登陆后需要填写必要信息，再然后就是创建项目了，如下图：

![image-20190420120205589](./img/image-20190420120205589.png)

如果是vue项目，看红框，创建后会进入一个简单的帮助页面主要代码如下：

```javascript
import Vue from 'vue'
import App from './App.vue'
import {init} from '@sentry/browser'
import * as Integrations from '@sentry/integrations';

Vue.config.productionTip = false

init({
  release: '1.0.0',
  dsn: '',
  integrations: [
    new Integrations.Vue({ 
    Vue,
    attachProps: true
  })]
})
new Vue({
  render: h => h(App),
}).$mount('#app')
```

需要安装`@sentry/browser` `@sentry/integrations`两个sentry npm包。这里需要主要dsn的配置，

`{PROTOCOL}://{PUBLIC_KEY}:{SECRET_KEY}@{HOST}/{PATH}{PROJECT_ID}`

我是本地搭建的环境，在创建的项目设置里可以获取，`客户端密钥 (DSN)` 里点击配置，页面下面有如下：

![image-20190420121300508](./img/image-20190420121300508.png)

根基这三个即可组成dsn。例如`http://7ff913e92bc94b8eb2497c55a1974e38:9c0a7f751710403398c373a9e8ef4ea5@sentry.cool.com/3` ，这个链接仅仅作为参考。然后我们在代码中故意写异常代码，比如：

![image-20190420121619255](./img/image-20190420121619255.png)

yarn build后我们在vue的dist目录里运行一个web server，我推荐`http-server` ，用起来简单，npm安装即可。

如果是点击事件点用上图的show方法，我们即可看到sentry后台的错误统计。如下：

![image-20190420121922802](./img/image-20190420121922802.png)

进入其中一个错误的页面如下：

![image-20190420122015961](./img/image-20190420122015961.png)

信息比较丰富了，但是可以发现，报错很不友好，所以需要sourcemap的支持了。下面我们就是要上传sourcemap文件到sentry。

##### 3、sourcemap

这里需要sentry-cli，需要npm安装。由于sentry运行在docker容器内，如果想要在宿主机器运行创建等操作需要一个配置文件，当然在sentry-cli命令参数后指定也可，只是这样会导致命令太长。`~/.sentryclirc` 我们只需要常见这个文件即可。内容如下：

```ini
[defaults]
org=sentry
project=vue-demo
url=http://sentry.yourdemo.com

[auth]
token=2323
```

token在sentry web后台中，如下：

![image-20190420123701117](./img/image-20190420123701117.png)

然后创建release 1.0.0：

```bash
sentry-cli --log-level=debug releases --org sentry new 1.0.0
```

然后进入vue build后的dist目录下运行：

```bash
sentry-cli --log-level=debug releases -o sentry files 1.0.0 upload-sourcemaps ./js  --url-prefix http://127.0.0.1:8082/js --rewrite
```

这里要说的是`./js` 里面要有打包后的js以及sourcemap。--url-prefix 后面是vue web站点请求js的路径的path了。随后`sentry-cli releases finalize 1.0.0`。

没意外的话，在触发几次报错，看看错误信息即可知道sourcemap是否有效了。如图：

![image-20190420124618017](./img/image-20190420124618017.png)

如上图，可以看到我们可以定位到具体的组件，以及哪一行代码。

就写这么多吧。