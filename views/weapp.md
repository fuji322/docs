# 在微信小程序中使用 {% if node == 'qcloud' %}TAB{% else %}LeanCloud{% endif %}

微信小程序是一个全新的跨平台移动应用平台，LeanCloud 为小程序提供一站式后端云服务，为你免去服务器维护、证书配置等繁琐的工作，大幅降低你的开发和运维成本。本文说明了如何在微信小程序中使用 LeanCloud 提供的各项服务。

## Demo
我们在小程序上实现了 LeanTodo 应用。在这个 Demo 中你可以看到：

- 如何对云端数据进行查询、增加、修改与删除
- 如何将查询结果数组绑定到视图层进行展示，以及如何在点击事件中得到对应的数组项
- 如何自动登录 LeanCloud 用户系统，以及如何在登录后设置帐号与密码以供用户在其他平台的 LeanTodo 应用上登录
- 如何集成微信支付

你可以通过微信扫描以下二维码进入 Demo。 Demo 的源码与运行说明请参考 [https://github.com/leancloud/leantodo-weapp](https://github.com/leancloud/leantodo-weapp)。

<img src="images/leantodo-weapp-qr.png" alt="LeanTodo Weapp QR" width="250">

## 准备工作
### 创建应用
- 如果你还没有创建过 LeanCloud 应用，请登录 LeanCloud 控制台 [创建一个新应用](/applist.html#/newapp)。
- 如果你还没有小程序帐号，请访问 [微信公众平台](https://mp.weixin.qq.com) 注册一个小程序帐号。如果你不需要进行真机调试可以跳过这一步。
- 下载 [小程序开发工具](https://mp.weixin.qq.com/debug/wxadoc/dev/devtools/download.html)，按照 [小程序开发教程](https://mp.weixin.qq.com/debug/wxadoc/dev/) 创建一个项目。

### 配置域名白名单
请按照 [小程序域名白名单配置](weapp-domains.html) 的步骤配置。如果你不需要进行真机调试可以跳过这一步（可在开发者工具的项目配置中勾选「开发环境不校验请求域名」）。

## 存储

要使用 LeanCloud 的数据存储、用户系统、调用云引擎等功能，需要使用 LeanCloud 存储 SDK。

### 安装与初始化
2. 下载 [`av-weapp-min.js`](https://unpkg.com/leancloud-storage@^3.0.0-alpha/dist/av-weapp-min.js)（[镜像](https://raw.githubusercontent.com/leancloud/javascript-sdk/next-dist/dist/av-weapp-min.js)），移动到 `libs` 目录。
3. 在 `app.js` 中使用 `const AV = require('./libs/av-weapp-min.js');` 获得 `AV` 的引用。在其他文件中使用时请将路径替换成对应的相对路径。 
4. 在 `app.js` 中初始化应用： 
  ```javascript 
  AV.init({ 
    appId: '{{appid}}', 
    appKey: '{{appkey}}', 
  }); 
  ```

### 对象存储
所有的对象存储 API 都能正常使用，详细的用法请参考 [JavaScript 数据存储开发指南](leanstorage_guide-js.html)。

#### 数据绑定
直接使用 `this.setData()` 将 `AV.Object` 对象设置为当前页面的 data，即可在 WXML 中使用 Mustache 语法访问绑定的数据了。下面这个例子展示了如何将一个 Query 的查询结果显示在页面上：

```javascript
// pages/todos/todos.js
Page({
  data: {
    todos: [],
  },
  onReady: function() {
    new AV.Query('Todo')
      .descending('createdAt')
      .find()
      .then(todos => this.setData({ todos }))
      .catch(console.error);
  },
});
```

<pre ng-non-bindable><code class="lang-html">&lt;!-- pages/todos/todos.wxml --&gt;
&lt;block wx:for=&quot;&lbrace;&lbrace;todos&rbrace;&rbrace;&quot; wx:for-item=&quot;todo&quot; wx:key=&quot;objectId&quot;&gt;
  &lt;text data-id=&quot;&lbrace;&lbrace;todo.objectId}}&quot;&gt;
    &lbrace;&lbrace;todo.content}}
  &lt;/text&gt;
&lt;/block&gt;
</code></pre>

使用 `include` 得到的嵌套对象也可以直接在视图层通过 `.` 访问到：

```javascript
// pages/student/student.js
Page({
  data: {
    student: null,
  },
  onReady: function() {
    new AV.Query('Student')
      .include('avatar') // avatar is an AV.File
      .get('56a9803e1532bc005303650c')
      .then(student => this.setData({ student }))
      .catch(console.error);
  },
});
```

<pre ng-non-bindable><code class="lang-html">&lt;!-- pages/student/student.wxml --&gt;
&lt;image src="&lbrace;&lbrace;student.avatar.url}}&quot;&gt;&lt;/image&gt;
</code></pre>

### 文件存储

在小程序中，可以将用户相册或拍照得到的图片上传到 LeanCloud 服务器进行保存。首先通过 `wx.chooseImage` 方法选择或拍摄照片，得到本地临时文件的路径，然后按照下面的方法构造一个 `AV.File` 将其上传到 LeanCloud：

```javascript
wx.chooseImage({
  count: 1,
  sizeType: ['original', 'compressed'],
  sourceType: ['album', 'camera'],
  success: function(res) {
    var tempFilePath = res.tempFilePaths[0];
    new AV.File('file-name', {
      blob: {
        uri: tempFilePath,
      },
    }).save().then(
      file => console.log(file.url())
    ).catch(console.error);
  }
});
```

上传成功后可以通过 `file.url()` 方法得到服务端的图片 url。

#### 文件批量上传
目前在小程序 Android 上不支持文件并发上传，需要串行上传：

```javascript
res.tempFilePaths.map(tempFilePath => () => new AV.File('filename', {
  blob: {
    uri: tempFilePath,
  },
}).save()).reduce(
  (m, p) => m.then(v => AV.Promise.all([...v, p()])),
  AV.Promise.resolve([])
).then(files => console.log(files.map(file => file.url()))).catch(console.error);
```

### 用户系统

小程序中提供了登录 API 来获取微信的用户登录状态，应用可以访问到用户的昵称、性别等基本信息，但是如果想要保存额外的用户信息，如用户的手机号码、收货地址等，则需要使用 LeanCloud 的用户系统。

#### 一键登录
LeanCloud 的用户系统现已支持一键使用微信用户身份登录。要使用一键登录功能，需要先设置小程序的 AppID 与 AppSecret：

1. 登录 [微信公众平台](https://mp.weixin.qq.com)，在 **设置** > **开发设置** 中获得 AppID 与 AppSecret。
2. 前往 LeanCloud 控制台 > **组件** > **社交**，保存「微信小程序」的 AppID 与 AppSecret。

现在，你可以在应用中使用 `AV.User.loginWithWeapp()` 方法来使用当前用户身份登录了。

```javascript
AV.User.loginWithWeapp().then(user => {
  this.globalData.user = user.toJSON();
}).catch(console.error);
```

如果该用户是第一次使用此应用，调用登录 API 会创建一个新的用户，你可以在 控制台 > **存储** 中的 `_User` 表中看到该用户的信息，如果该用户曾经使用该方式登录过此应用，再次调用登录 API 会返回同一个用户。

用户的登录状态会保存在客户端中，可以使用 `AV.User.current()` 方法来获取当前登录的用户，下面的例子展示了如何同步登录用户的信息：

```javascript
// 假设已经通过 AV.User.loginWithWeapp() 登录
// 获得当前登录用户
const user = AV.User.current();
// 调用小程序 API，得到用户信息
wx.getUserInfo({
  success: ({userInfo}) => {
    // 更新当前用户的信息
    user.set(userInfo).save().then(user => {
      // 成功，此时可在控制台中看到更新后的用户信息
      this.globalData.user = user.toJSON();
    }).catch(console.error);
  }
});
```

使用一键登录方式登录时，LeanCloud 会将该用户的小程序 openid 保存在对应的 `user.authData.lc_weapp` 属性中，你可以在控制台的 _User 表中看到。该字段在客户端不可见，你可以使用 masterKey 在云引擎中获取该用户的 openid 进行支付、推送等操作。详情请参考 [支付](#支付)。

#### 启用其他登录方式
由于 `AV.User.loginWithWeapp()` 只能在小程序中使用，所以使用该 API 创建的用户无法直接在小程序之外的平台上登录。如果需要使用 LeanCloud 用户系统提供的其他登录方式，如用手机号验证码登录、邮箱密码登录等，在小程序一键登录后设置对应的用户属性即可：

```javascript
// 小程序登录
AV.User.loginWithWeapp().then(user => {
  // 设置并保存手机号
  user.setMobilePhoneNumber('13000000000');
  return user.save();
}).then(user => {
  // 发送验证短信
  return AV.User.requestMobilePhoneVerify(user.getMobilePhoneNumber());
}).then({
  // 用户填写收到短信验证码后再调用 AV.User.verifyMobilePhone(code) 完成手机号的绑定
  // 成功后用户的 mobilePhoneVerified 字段会被置为 true
  // 此后用户便可以使用手机号加动态验证码登录了
}).catch(console.error);
```

<div class="callout callout-info">
验证手机号码功能要求在控制台的应用设置中启用「用户注册时，向注册手机号码发送验证短信」。
</div>

#### 绑定现有用户
如果你的应用已经在使用 LeanCloud 的用户系统，或者用户已经通过其他方式注册了你的应用（比如在 Web 端通过用户名密码注册），可以通过在小程序中调用 `AV.User#linkWithWeapp()` 来关联已有的账户：

```javascript
// 首先，使用用户名与密码登录一个已经存在的用户
AV.User.logIn('username', 'password').then(user => {
  // 将当前的微信用户与当前登录用户关联
  return user.linkWithWeapp();
}).catch(console.error);
```

### 云引擎
使用云引擎可以方便地实现一些在服务器执行的逻辑，比如处理敏感信息、数据聚合、采集数据等，并且不需要准备额外的服务器。

SDK 所有的云引擎相关的 API 都能正常使用，详细的用法请参考 [云函数开发指南](leanengine_cloudfunction_guide-node.html)。

使用云引擎实现小程序支付的方案参见 [支付](#支付)。

## 实时通讯

要使用 LeanCloud 的聊天、实时消息功能，需要使用 LeanCloud 实时通讯 SDK。

### 安装与初始化
2. 下载 [`realtime.weapp.min.js`](https://unpkg.com/leancloud-realtime@^3.3.0/dist/realtime.weapp.min.js)（[镜像](https://raw.githubusercontent.com/leancloud/js-realtime-sdk/dist/dist/realtime.weapp.min.js)），移动到 `libs` 目录。
3. 在 `app.js` 中使用 `const Realtime = require('./libs/realtime.weapp.min.js').Realtime;` 获得 `Realtime` 的引用。在其他文件中使用时请将路径替换成对应的相对路径。
4. 在 `app.js` 中初始化应用：
  ```javascript
  const realtime = new Realtime({
    appId: '{{appid}}',
    noBinary: true,
  });
  ```

实时通讯 SDK 的详细用法请参考 [实时通信开发指南](realtime_guide-js.html)。

### 富媒体消息
要在小程序中使用实时通讯 SDK 的富媒体消息插件，有一些额外的约束：

1. 安装存储 SDK 至 `libs` 目录，并将文件重命名为 `leancloud-storage.js`。
2. 安装实时通讯 SDK 至 `libs` 目录，并将文件重命名为 `leancloud-realtime.js`。
3. 下载 [`leancloud-realtime-plugin-typed-messages.js`](https://unpkg.com/leancloud-realtime-plugin-typed-messages@^1.0.0)，移动到 `libs` 目录。必须保证<u>三个文件在同一目录中</u>。
4. 在 `app.js` 中<u>依次加载</u> `leancloud-storage.js`、`leancloud-realtime.js` 和 `leancloud-realtime-plugin-typed-messages.js`。
  ```javascript
  const AV = require('./libs/leancloud-storage.js');
  const Realtime = require('./libs/leancloud-realtime.js').Realtime;
  const TypedMessagesPlugin = require('./libs/leancloud-realtime-plugin-typed-messages.js').TypedMessagesPlugin;
  const ImageMessage = require('./libs/leancloud-realtime-plugin-typed-messages.js').ImageMessage;
  ```
5. 在 `app.js` 中初始化应用：
  ```javascript
  // 初始化存储 SDK
  AV.init({
    appId: '{{appid}}',
    appKey: '{{appkey}}',
  });
  // 初始化实时通讯 SDK
  const realtime = new Realtime({
    appId: '{{appid}}',
    noBinary: true,
    plugins: [TypedMessagesPlugin], // 注册富媒体消息插件
  });
  ```

富媒体消息的用法请参考 [实时通信开发指南 - 富媒体消息](realtime_guide-js.html#富媒体消息)。

## 支付

### 配置

在开始之前，请确保已经在微信小程序后台开启了「微信支付」功能，然后按照下面的步骤配置云引擎环境变量：

1. 进入应用控制台 - 云引擎 - 设置
2. 设置应用的二级域名并保存
3. 添加并保存以下环境变量
  - `WEIXIN_APPID`：小程序 AppId
  - `WEIXIN_MCHID`：微信支付商户号
  - `WEIXIN_PAY_SECRET`：微信支付 API 密钥（[微信商户平台](https://pay.weixin.qq.com) - 账户设置 - API安全 - 密钥设置）
  - `WEIXIN_NOTIFY_URL`：`https://{{yourdomain}}.leanapp.cn/weixin/pay-callback`，其中 `yourdomain` 是第二步中设置的二级域名

<details>
<summary>查看示例</summary>
<p>
![image](https://cloud.githubusercontent.com/assets/175227/22236906/7c651c80-e243-11e6-819b-007d5862bdbf.png)
</p>
</details>

### 服务端开发

首先确认本机已经安装 [Node.js](http://nodejs.org/) 运行环境和 [LeanCloud 命令行工具](leanengine_cli.html)，然后执行下列指令下载示例项目：

```
$ git clone https://github.com/leancloud/weapp-pay-getting-started.git
$ cd weapp-pay-getting-started
```

安装依赖：

```
npm install
```

登录并关联应用：

```
lean login
lean checkout
```

启动项目：

```
lean up
```

之后你就可以在 [localhost:3001](http://localhost:3001) 调试云函数了。

示例项目中与支付直接相关代码有三部分：

* `order.js`：对应 Order 表，定义了部分字段的 getter/setter，以及 `place` 方法用于向微信 API 提交订单。
* `cloud.js`：其中定义了名为 `order` 的云函数，这个云函数会获取当前用户的 `openid`，以其身份创建了一个 1 分钱的 order 并下单，最后返回签名过的订单信息。
* `routers/weixin.js`：其中定义了 `pay-callback` 的处理函数，当用户支付成功后微信调用这个 URL，这个函数将对应的订单状态更新为 `SUCCESS`。

请根据你的业务需要修改代码。参考文档：

* [微信支付统一下单 API 参数与错误码](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_1)
* [微信支付结果通知参数](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_7)

完成开发后部署到预备环境（若无预备环境则直接部署到生产环境）：
```
lean deploy
```

### 客户端开发

客户端完成一次支付需要分两步：

1. 用户登录后，调用名为 `order` 的云函数下单，返回签名过的订单信息。
2. 调用支付 API（`wx.requestPayment`），传入上一步返回的订单信息，发起支付。

```javascript
Cloud.run('order').then((data) => {
  data.success = () => {
    // 支付成功
  }
  data.fail = ({ errMsg }) => {
    // 错误处理
  });
  wx.requestPayment(data);
}).catch(error => {
  // 错误处理
})
```

客户端的示例代码参见 [Demo](https://github.com/leancloud/leantodo-weapp) 打赏功能。参考文档：

* [小程序客户端发起支付 API](https://mp.weixin.qq.com/debug/wxadoc/dev/api/api-pay.html)

## 反馈
如果在微信小程序中使用 LeanCloud 时遇到问题，欢迎通过我们的 [论坛](https://forum.leancloud.cn/c/jing-xuan-faq/weapp) 进行反馈。
