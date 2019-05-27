---
id: "2018-10-01-13-58"
date: "2018/10/01 13:58"
title: "node,vue开发教学管理系统"
tags: ["node", "vue","element-ui","axios","koa","redis","mysql","jwt"]
categories: 
- "node"
- "项目"
---

&emsp;&emsp;毕业才刚刚两个多月而已，现在想想大学生活是已经是那么的遥不可及，感觉已经过了好久好久，社会了两个月才明白学校的好啊。。。额，扯远了，自从毕业开始就想找个时间写下毕设的记录总结，结果找了好久好久到今天才开始动笔。

&emsp;&emsp;我的毕业设计题目是：教学辅助系统的设计与实现，，是不是很俗。。。至于为啥是这个题目呢，完全是被导师坑了。。。。。

demo地址：ali.tapme.top:8008 123456/123456

## 1、需求分析

&emsp;&emsp;拿到这个题目想着这个可能被做了无数次了，就像着哪里能够做出点创新，，最后强行创新出了一个个性化组题（根据学生水平出题）和徽章激励（达到某个要求给予一个徽章）。最后就产生了如下需求，系统有学生端和管理端：

学生端：

- 个人资料设置
- 徽章激励机制
- 查看课程信息，下载课程资料
- 知识点检测及针对性训练
- 在线作业，考试
- 在线答疑，向老师或者学生提问

管理端：

- 课程管理，用户管理（需要管理员权限）
- 课程信息管理
- 课程公告管理
- 题库管理，支持单选，多选，填空，编程题，支持题目编组
- 发布作业，包括个性组题和手动组题
- 发布考试，包括随机出题和手动出题
- 自动判题，支持编程题判重
- 在线答疑，给学生解答
- 统计分析，包含测试统计和课程统计

洋洋洒洒需求列了一大堆，后面才发现是给自己挖坑，，答辩老师一看这类的题目就不感兴趣了，不论你做的咋样（况且我的演讲能力真的很一般），最后累死累活写了一大堆功能也没太高的分，，不过倒是让我的系统设计能力和代码能力有了不少的提高。

<!-- more -->

## 2、架构选择

&emsp;&emsp;大三的时候了解到 Node.js 这个比较“奇葩"的异步语言，再加上在公司实习了三个月也是用的 node 开发，对 node 已经比较熟悉了，于是就用它做了后台，前端用最近比较火的 vue.js 做单页应用。当时还想着负载均衡啥的，就没有用传统的 session，cookie 机制，转而用 jwt 做的基于 token 的身份认证，同时后台接口也是类 Restful 风格的（因为纯正的 Rest 接口太难设计了）。

总的来说后台用了以下技术和框架：

&emsp;&emsp;总的来说后台用了以下技术和框架：

- 语言：Node.js
- web 框架：kOA
- 前后台传输协议：jwt
- 缓存：redis
- 数据库：mysql
- 编程题判题核心：[青岛大学 OJ 判题核心](https://github.com/QingdaoU/JudgeServer)
- 代码判重：[SIM](https://dickgrune.com/Programs/similarity_tester/)

前台技术如下：

- 框架：Vue.js
- UI 框架：Element-UI
- 图表组件：G2

## 3、系统基础框架搭建

&emsp;&emsp;本系统是前后端分离的，下面分别介绍前后端的实现基础。

### 1、后台

&emsp;&emsp;一个 web 后台最重要的无非那么几个部分：路由；权限验证；数据持久化。

#### a、路由

KOA 作为一个 web 框架其实它本身并没有提供路由功能，需要配合使用 koa-router 来实现路由，koa-router 以类似下面这样的风格来进行路由：

&emsp;&emsp;KOA 作为一个 web 框架其实它本身并没有提供路由功能，需要配合使用 koa-router 来实现路由，koa-router 以类似下面这样的风格来进行路由：

```javascript
const app = require('koa');
const router = require('koa-router');
router.get('/hello', koa => {
  koa.response = 'hello';
});
app.use(router.routes());
```

显然这样在项目中是很不方便的，如果每个路由都要手动进行挂载，很难将每个文件中的路由都挂载到一个 router 中。因此在参考网上的实现后，我写了一个方法在启动时自动扫描某个文件夹下所有的路由文件并挂载到 router 中，代码如下：

```javascript
const fs = require('fs');
const path = require('path');
const koaBody = require('koa-body');
const config = require('../config/config.js');

function addMapping(router, filePath) {
  let mapping = require(filePath);
  for (let url in mapping) {
    if (url.startsWith('GET ')) {
      let temp = url.substring(4);
      router.get(temp, mapping[url]);
      console.log(`----GET：${temp}`);
    } else if (url.startsWith('POST ')) {
      let temp = url.substring(5);
      router.post(temp, mapping[url]);
      console.log(`----POST：${temp}`);
    } else if (url.startsWith('PUT ')) {
      let temp = url.substring(4);
      router.put(temp, mapping[url]);
      console.log(`----PUT：${temp}`);
    } else if (url.startsWith('DELETE ')) {
      let temp = url.substring(7);
      router.delete(temp, mapping[url]);
      console.log(`----DELETE: ${temp}`);
    } else {
      console.log(`xxxxx无效路径：${url}`);
    }
  }
}

function addControllers(router, filePath) {
  let files = fs.readdirSync(filePath);
  files.forEach(element => {
    let temp = path.join(filePath, element);
    let state = fs.statSync(temp);
    if (state.isDirectory()) {
      addControllers(router, temp);
    } else {
      if (!temp.endsWith('Helper.js')) {
        console.log('\n--开始处理: ' + element + '路由');
        addMapping(router, temp);
      }
    }
  });
}

function engine(router, folder) {
  addControllers(router, folder);
  return router.routes();
}

module.exports = engine;
```

然后在 index.js 中 use 此方法：

```
const RouterMW = require("./middleware/controllerEngine.js");
app.use(RouterMW(router,path.join(config.rootPath, 'api')));
```

然后路由文件以下面的形式编写：

```javascript
const knowledgePointDao = require('../dao/knowledgePointDao.js');

/**
 * 返回某门课的全部知识点,按章节分类
 */
exports['GET /course/:c_id/knowledge_point'] = async (ctx, next) => {
  let res = await knowledgePointDao.getPontsOrderBySection(ctx.params.c_id);
  ctx.onSuccess(res);
};

//返回某位学生知识点答题情况
exports['GET /user/:u_id/course/:c_id/knowledge_point/condition'] = async (
  ctx,
  next
) => {
  let { u_id, c_id } = ctx.params;
  let res = await knowledgePointDao.getStudentCondition(u_id, c_id);
  ctx.onSuccess(res);
};
```

#### b、权限验证

&emsp;&emsp;权限管理是一个系统最重要的部分之一，目前主流的方式为**基于角色的权限管理**， 一个用户对应多个角色，每个角色对应多个权限（本系统中每个用户对应一个身份，每个身份对应多个角色）。我们的系统如何实现的呢？先从登录开始说起，本系统抛弃了传统的 cookie，session 模式，使用 json web token（JWT）来做身份认证，用户登录后返回一个 token 给客户端，代码如下所示：

```javascript
//生成随机盐值
let str = StringHelper.getRandomString(0, 10);
//使用该盐值生成token
let token = jwt.sign(
  {
    u_id: userInfo.u_id,
    isRememberMe
  },
  str,
  {
    expiresIn: isRememberMe
      ? config.longTokenExpiration
      : config.shortTokenExpiration
  }
);
//token-盐值存入redis，如想让该token过期，redis中清楚该token键值对即可
await RedisHelper.setString(token, str, 30 * 24 * 60 * 60);
res.code = 1;
res.info = '登录成功';
res.data = {
  u_type: userInfo.u_type,
  u_id: userInfo.u_id,
  token
};
```

以后每次客户端请求都要在 header 中设置该 token，然后每次服务端收到请求都先验证是否拥有权限，验证代码使用`router.use(auth)`,挂载到 koa-router 中，这样每次在进入具体的路由前都要先执行 auth 方法进行权限验证,主要验证代码逻辑如下：

```javascript
/**
 * 1 验证成功
 * 2 登录信息无效 401
 * 3 已登录，无操作权限 403
 * 4 token已过期
 */
let verify = async ctx => {
  let token = ctx.headers.authorization;
  if (typeof token != 'string') {
    return 2;
  }
  let yan = await redisHelper.getString(token);
  if (yan == null) {
    return 2;
  }
  let data;
  try {
    data = jwt.verify(token, yan);
  } catch (e) {
    return 2;
  }
  if (data.exp * 1000 < Date.now()) {
    return 4;
  }
  //判断是否需要刷新token，如需要刷新将新token写入响应头
  if (!data.isRememberMe && data.exp * 1000 - Date.now() < 30 * 60 * 1000) {
    //token有效期不足半小时，重新签发新token给客户端
    let newYan = StringHelper.getRandomString(0, 10);
    let newToken = jwt.sign(
      {
        u_id: data.u_id,
        isRememberMe: false
      },
      newYan,
      {
        expiresIn: config.shortTokenExpiration
      }
    );
    // await redisHelper.deleteKey(token);
    await redisHelper.setString(newToken, newYan, config.shortTokenExpiration);
    ctx.response.set('new-token', newToken);
    ctx.response.set('Access-Control-Expose-Headers', 'new-token');
  }
  //获取用户信息
  let userInfoKey = data.u_id + '_userInfo';
  let userInfo = await redisHelper.getString(userInfoKey);
  if (userInfo == null || Object.keys(userInfo).length != 3) {
    userInfo = await mysqlHelper.first(
      `select u_id,u_type,j_id from user where u_id=?`,
      data.u_id
    );
    await redisHelper.setString(
      userInfoKey,
      JSON.stringify(userInfo),
      24 * 60 * 60
    );
  } else {
    userInfo = JSON.parse(userInfo);
  }
  ctx.userInfo = userInfo;
  //更新用户上次访问时间
  mysqlHelper.execute(
    `update user set last_login_time=? where u_id=?`,
    Date.now(),
    userInfo.u_id
  );
  //管理员拥有全部权限
  if (userInfo.u_type == 0) {
    return 1;
  }
  //获取该用户类型权限
  let authKey = userInfo.j_id + '_authority';
  let urls = await redisHelper.getObject(authKey);
  // let urls = null;
  if (urls == null) {
    urls = await mysqlHelper.row(
      `
            select b.r_id,b.url,b.method from jurisdiction_resource a inner join resource b on a.r_id = b.r_id where a.j_id=?
            `,
      userInfo.j_id
    );
    let temp = {};
    urls.forEach(item => {
      temp[item.url + item.method] = true;
    });
    await redisHelper.setObject(authKey, temp);
    urls = temp;
  }
  //判断是否拥有权限
  if (
    urls.hasOwnProperty(
      ctx._matchedRoute.replace(config.url_prefix, '') + ctx.method
    )
  ) {
    return 1;
  } else {
    return 3;
  }
};
```

根据用户 id 获取用户身份 id，根据用户身份 id 从 redis 中获取拥有的权限，如为 null，从 mysql 数据库中拉取，并存入 redis 中，然后判断是否拥有要访问的 url 权限。

#### c、数据持久化

&emsp;&emsp;本系统中使用 mysql 存储数据，redis 做缓存，由于当时操作库不支持 promise，故对它两做了个 promise 封装，方便代码中调用，参见：[MysqlHelper](https://github.com/FleyX/teach_system/tree/master/teachSystem/util/MysqlHelper.js),[RedisHelper.js](https://github.com/FleyX/teach_system/tree/master/teachSystem/util/RedisHelper.js)。

### 2、前端

&emsp;&emsp;前端使用 vue-cli 构建 vue 项目，主要用到了 vue-router,element-ui,axios 这三个组件。

#### a、路由组织

&emsp;&emsp;单页应用需要前端自己组织路由。本系统将路由分成了三个部分：公共，管理端，学生端。index.js 如下：

```javascript
export default new Router({
  mode: 'history',
  base: '/app/',
  routes: [
    {
      path: '',
      name: 'indexPage',
      component: IndexPage
    },
    {
      path: '/about',
      name: 'about',
      component: About
    },
    Admin,
    Client,
    Public,
    {
      path: '*',
      name: 'NotFound',
      component: NotFound
    }
  ]
});
```

其中的 Admin，Client，Public 分别为各部分的路由，以子路由的形式一级级组织。如下所示：

```javascript
export default {
  path: "/client",
  component: Client,
  beforeEnter: (to, from, next) => {
    if (getClientUserInfo() == null) {
      next({
        path: '/public/client_login',
        replace: true,
      })
    } else {
      next();
    }
  },
  children: [{
      //学生端主页
      path: '',
      name: "ClientMain",
      component: ClientHome
    }, {
      //学生个人资料页面
      path: 'person/student_info',
      name: "StudentInfo",
      component: StudentInfo
    }, {
      //公告页面
      path: 'course/:c_id/announcement',
      name: 'Main',
      component: Announcement
    }, {
      //课程基本信息
      path: 'course/:c_id/base',
      component: ClientMain,
      children: [{
        path: 'course_intro',
        name: "ClientCourseIntro",
        component: CourseIntro
      }, {
        path: 'exam_type',
        name: "ClientExamType",
        component: ExamType
      }
      ......
```

其中的 beforEnter 为钩子函数，每次进入路由时执行该函数，用于判断用户是否登录。这里涉及到了一个前端鉴权的概念，由于前后端分离了，前端也必须做鉴权以免用户进入到了无权限的页面，这里我只是简单的做了登录判断，更详细的 url 鉴权也可实现，只需在对应的钩子函数中进行鉴权操作，更多关于钩子函数信息[点击这里](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html)。

#### b、请求封装

&emsp;&emsp;前端还有一个比较重要的部分是 ajax 请求的处理，请求处理还保护错误处理，有些错误只需要统一处理，而有些又需要独立的处理，这样一来就需要根据业务需求进行一下请求封装了，对结果进行处理后再返回给调用者。我的实现思路是发起请求，收到响应后先对错误进行一个同意弹窗提示，然后再将错误继续向后传递，调用者可选择性的捕获错误进行针对性处理，主要代码如下：

```javascript
request = (url, method, params, form, isFormData, type) => {
  let token;
  if (type == 'admin') token = getToken();
  else token = getClientToken();
  let headers = {
    Authorization: token
  };
  if (isFormData) {
    headers['Content-Type'] = 'multipart/form-data';
  }
  return new Promise((resolve, reject) => {
    axios({
      url,
      method,
      params,
      data: form,
      headers
      // timeout:2000
    })
      .then(res => {
        resolve(res.data);
        //检查是否有更新token
        // console.log(res);
        if (res.headers['new-token'] != undefined) {
          console.log('set new token');
          if (vm.$route.path.startsWith('/admin')) {
            localStorage.setItem('token', res.headers['new-token']);
            window.token = undefined;
          } else if (vm.$route.path.startsWith('/client')) {
            localStorage.setItem('clientToken', res.headers['new-token']);
            window.clientToken = undefined;
          }
        }
      })
      .catch(err => {
        reject(err);
        if (err.code == 'ECONNABORTED') {
          alertNotify('错误', '请求超时', 'error');
          return;
        }
        if (err.message == 'Network Error') {
          alertNotify('错误', '无法连接服务器', 'error');
          return;
        }
        if (err.response != undefined) {
          switch (err.response.status) {
            case 401:
              if (window.isGoToLogin) {
                return;
              }
              //使用该变量表示是否已经弹窗提示了，避免大量未登录弹窗堆积。
              window.isGoToLogin = true;
              vm.$alert(err.response.data, '警告', {
                type: 'warning',
                showClose: false
              }).then(res => {
                window.isGoToLogin = false;
                if (vm.$route.path.startsWith('/admin/')) {
                  clearInfo();
                  vm.$router.replace('/public/admin_login');
                } else {
                  clearClientInfo();
                  vm.$router.replace('/public/client_login');
                }
              });
              break;
            case 403:
              alertNotify(
                'Error:403',
                '拒绝执行：' + err.response.data,
                'error'
              );
              break;
            case 404:
              alertNotify(
                'Error:404',
                '找不到资源：' + url.substr(0, url.indexOf('?')),
                'error'
              );
              break;
            case 400:
              alertNotify(
                'Error:400',
                '请求参数错误：' + err.response.data,
                'error'
              );
              break;
            case 500:
              alertNotify(
                'Error:500',
                '服务器内部错误：' + err.response.data,
                'error'
              );
            default:
              console.log('存在错误未处理：' + err);
          }
        } else {
          console.log(err);
        }
      });
  });
};
```

&emsp;&emsp;到这里就算是简单介绍完了,,想要更加深入了解的可以去 github 查看源代码，地址如下：[https://github.com/FleyX/teach_system，](https://github.com/FleyX/teach_system)记得 star 哦！
