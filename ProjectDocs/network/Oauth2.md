# OAuth2.0

OAuth 2.0是目前最流行的授权机制，用来授权第三方应用，获取用户数据。

这个标准比较抽象，使用了很多术语，初学者不容易理解。其实说起来并不复杂，下面我就通过一个简单的类比，帮助大家轻松理解，OAuth 2.0 到底是什么。

## OAuth 2.0 的一个简单解释

### 一、快递员问题

我住在一个大型的居民小区。

![](../images/network/auth/bg2019040401.jpg)

小区有门禁系统。

![](../images/network/auth/bg2019040402.jpg)

进入的时候需要输入密码。

![](../images/network/auth/bg2019040403.jpg)

我经常网购和外卖，每天都有快递员来送货。我必须找到一个办法，让快递员通过门禁系统，进入小区。

![](../images/network/auth/bg2019040404.jpg)

如果我把自己的密码，告诉快递员，他就拥有了与我同样的权限，这样好像不太合适。万一我想取消他进入小区的权力，也很麻烦，我自己的密码也得跟着改了，还得通知其他的快递员。

有没有一种办法，让快递员能够自由进入小区，又不必知道小区居民的密码，而且他的唯一权限就是送货，其他需要密码的场合，他都没有权限？

### 二、授权机制的设计

于是，我设计了一套授权机制。

第一步，门禁系统的密码输入器下面，增加一个按钮，叫做"获取授权"。快递员需要首先按这个按钮，去申请授权。

第二步，他按下按钮以后，屋主（也就是我）的手机就会跳出对话框：有人正在要求授权。系统还会显示该快递员的姓名、工号和所属的快递公司。

我确认请求属实，就点击按钮，告诉门禁系统，我同意给予他进入小区的授权。

第三步，门禁系统得到我的确认以后，向快递员显示一个进入小区的令牌（access token）。令牌就是类似密码的一串数字，只在短期内（比如七天）有效。

第四步，快递员向门禁系统输入令牌，进入小区。

有人可能会问，为什么不是远程为快递员开门，而要为他单独生成一个令牌？这是因为快递员可能每天都会来送货，第二天他还可以复用这个令牌。另外，有的小区有多重门禁，快递员可以使用同一个令牌通过它们。

### 三、互联网场景

我们把上面的例子搬到互联网，就是 OAuth 的设计了。

首先，居民小区就是储存用户数据的网络服务。比如，微信储存了我的好友信息，获取这些信息，就必须经过微信的"门禁系统"。

其次，快递员（或者说快递公司）就是第三方应用，想要穿过门禁系统，进入小区。

最后，我就是用户本人，同意授权第三方应用进入小区，获取我的数据。

**简单说，OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用进入系统，获取这些数据。系统从而产生一个短期的进入令牌（token），用来代替密码，供第三方应用使用。**

### 四、令牌与密码

令牌（token）与密码（password）的作用是一样的，都可以进入系统，但是有三点差异。

（1）令牌是短期的，到期会自动失效，用户自己无法修改。密码一般长期有效，用户不修改，就不会发生变化。

（2）令牌可以被数据所有者撤销，会立即失效。以上例而言，屋主可以随时取消快递员的令牌。密码一般不允许被他人撤销。

（3）令牌有权限范围（scope），比如只能进小区的二号门。对于网络服务来说，只读令牌就比读写令牌更安全。密码一般是完整权限。

上面这些设计，保证了令牌既可以让第三方应用获得权限，同时又随时可控，不会危及系统安全。这就是 OAuth 2.0 的优点。

注意，只要知道了令牌，就能进入系统。系统一般不会再次确认身份，所以**令牌必须保密，泄漏令牌与泄漏密码的后果是一样的。** 这也是为什么令牌的有效期，一般都设置得很短的原因。

OAuth 2.0 对于如何颁发令牌的细节，规定得非常详细。具体来说，一共分成四种授权类型（authorization grant），即四种颁发令牌的方式，适用于不同的互联网场景。

## OAuth 2.0 的四种方式

### RFC 6749

OAuth 2.0 的标准是 [RFC 6749](https://tools.ietf.org/html/rfc6749) 文件。该文件先解释了 OAuth 是什么。

> OAuth 引入了一个授权层，用来分离两种不同的角色：客户端和资源所有者。......资源所有者同意以后，资源服务器可以向客户端颁发令牌。客户端通过令牌，去请求数据。

这段话的意思就是，**OAuth 的核心就是向第三方应用颁发令牌。**然后，RFC 6749 接着写道：

> （由于互联网有多种场景，）本标准定义了获得令牌的四种授权方式（authorization grant ）。

也就是说，**OAuth 2.0 规定了四种获得令牌的流程。你可以选择最适合自己的那一种，向第三方应用颁发令牌。**下面就是这四种授权方式。

> -   授权码（authorization-code）
> -   隐藏式（implicit）
> -   密码式（password）：
> -   客户端凭证（client credentials）

注意，不管哪一种授权方式，第三方应用申请令牌之前，都必须先到系统备案，说明自己的身份，然后会拿到两个身份识别码：客户端 ID（client ID）和客户端密钥（client secret）。这是为了防止令牌被滥用，没有备案过的第三方应用，是不会拿到令牌的。

### 第一种授权方式：授权码

**授权码（authorization code）方式，指的是第三方应用先申请一个授权码，然后再用该码获取令牌。**

这种方式是最常用的流程，安全性也最高，它适用于那些有后端的 Web 应用。授权码通过前端传送，令牌则是储存在后端，而且所有与资源服务器的通信都在后端完成。这样的前后端分离，可以避免令牌泄漏。

第一步，A 网站提供一个链接，用户点击后就会跳转到 B 网站，授权用户数据给 A 网站使用。下面就是 A 网站跳转 B 网站的一个示意链接。

>     https://b.com/oauth/authorize?
>       response_type=code&
>       client_id=CLIENT_ID&
>       redirect_uri=CALLBACK_URL&
>       scope=read

上面 URL 中，`response_type`参数表示要求返回授权码（`code`），`client_id`参数让 B 知道是谁在请求，`redirect_uri`参数是 B 接受或拒绝请求后的跳转网址，`scope`参数表示要求的授权范围（这里是只读）。

![](../images/network/auth/bg2019040902.jpg)

第二步，用户跳转后，B 网站会要求用户登录，然后询问是否同意给予 A 网站授权。用户表示同意，这时 B 网站就会跳回`redirect_uri`参数指定的网址。跳转时，会传回一个授权码，就像下面这样。

>     https://a.com/callback?code=AUTHORIZATION_CODE

上面 URL 中，`code`参数就是授权码。

![](../images/network/auth/bg2019040907.jpg)

第三步，A 网站拿到授权码以后，就可以在后端，向 B 网站请求令牌。

>     https://b.com/oauth/token?
>      client_id=CLIENT_ID&
>      client_secret=CLIENT_SECRET&
>      grant_type=authorization_code&
>      code=AUTHORIZATION_CODE&
>      redirect_uri=CALLBACK_URL

上面 URL 中，`client_id`参数和`client_secret`参数用来让 B 确认 A 的身份（`client_secret`参数是保密的，因此只能在后端发请求），`grant_type`参数的值是`AUTHORIZATION_CODE`，表示采用的授权方式是授权码，`code`参数是上一步拿到的授权码，`redirect_uri`参数是令牌颁发后的回调网址。

![](../images/network/auth/bg2019040904.jpg)

第四步，B 网站收到请求以后，就会颁发令牌。具体做法是向`redirect_uri`指定的网址，发送一段 JSON 数据。

>     {    
>       "access_token":"ACCESS_TOKEN",
>       "token_type":"bearer",
>       "expires_in":2592000,
>       "refresh_token":"REFRESH_TOKEN",
>       "scope":"read",
>       "uid":100101,
>       "info":{...}
>     }

上面 JSON 数据中，`access_token`字段就是令牌，A 网站在后端拿到了。

![](../images/network/auth/bg2019040905.jpg)

### 第二种方式：隐藏式

有些 Web 应用是纯前端应用，没有后端。这时就不能用上面的方式了，必须将令牌储存在前端。**RFC 6749 就规定了第二种方式，允许直接向前端颁发令牌。这种方式没有授权码这个中间步骤，所以称为（授权码）"隐藏式"（implicit）。**

第一步，A 网站提供一个链接，要求用户跳转到 B 网站，授权用户数据给 A 网站使用。

>     https://b.com/oauth/authorize?
>       response_type=token&
>       client_id=CLIENT_ID&
>       redirect_uri=CALLBACK_URL&
>       scope=read

上面 URL 中，`response_type`参数为`token`，表示要求直接返回令牌。

第二步，用户跳转到 B 网站，登录后同意给予 A 网站授权。这时，B 网站就会跳回`redirect_uri`参数指定的跳转网址，并且把令牌作为 URL 参数，传给 A 网站。

>     https://a.com/callback#token=ACCESS_TOKEN

上面 URL 中，`token`参数就是令牌，A 网站因此直接在前端拿到令牌。

注意，令牌的位置是 URL 锚点（fragment），而不是查询字符串（querystring），这是因为 OAuth 2.0 允许跳转网址是 HTTP 协议，因此存在"中间人攻击"的风险，而浏览器跳转时，锚点不会发到服务器，就减少了泄漏令牌的风险。

![](../images/network/auth/bg2019040906.jpg)

这种方式把令牌直接传给前端，是很不安全的。因此，只能用于一些安全要求不高的场景，并且令牌的有效期必须非常短，通常就是会话期间（session）有效，浏览器关掉，令牌就失效了。

### 第三种方式：密码式

**如果你高度信任某个应用，RFC 6749 也允许用户把用户名和密码，直接告诉该应用。该应用就使用你的密码，申请令牌，这种方式称为"密码式"（password）。**

第一步，A 网站要求用户提供 B 网站的用户名和密码。拿到以后，A 就直接向 B 请求令牌。

>     https://oauth.b.com/token?
>       grant_type=password&
>       username=USERNAME&
>       password=PASSWORD&
>       client_id=CLIENT_ID

上面 URL 中，`grant_type`参数是授权方式，这里的`password`表示"密码式"，`username`和`password`是 B 的用户名和密码。

第二步，B 网站验证身份通过后，直接给出令牌。注意，这时不需要跳转，而是把令牌放在 JSON 数据里面，作为 HTTP 回应，A 因此拿到令牌。

这种方式需要用户给出自己的用户名/密码，显然风险很大，因此只适用于其他授权方式都无法采用的情况，而且必须是用户高度信任的应用。

### 第四种方式：凭证式

**最后一种方式是凭证式（client credentials），适用于没有前端的命令行应用，即在命令行下请求令牌。**

第一步，A 应用在命令行向 B 发出请求。

>     https://oauth.b.com/token?
>       grant_type=client_credentials&
>       client_id=CLIENT_ID&
>       client_secret=CLIENT_SECRET

上面 URL 中，`grant_type`参数等于`client_credentials`表示采用凭证式，`client_id`和`client_secret`用来让 B 确认 A 的身份。

第二步，B 网站验证通过以后，直接返回令牌。

这种方式给出的令牌，是针对第三方应用的，而不是针对用户的，即有可能多个用户共享同一个令牌。

#### 令牌的使用

A 网站拿到令牌以后，就可以向 B 网站的 API 请求数据了。

此时，每个发到 API 的请求，都必须带有令牌。具体做法是在请求的头信息，加上一个`Authorization`字段，令牌就放在这个字段里面。

>     curl -H "Authorization: Bearer ACCESS_TOKEN" \
>     "https://api.b.com"

上面命令中，`ACCESS_TOKEN`就是拿到的令牌。

#### 更新令牌

令牌的有效期到了，如果让用户重新走一遍上面的流程，再申请一个新的令牌，很可能体验不好，而且也没有必要。OAuth 2.0 允许用户自动更新令牌。

具体方法是，B 网站颁发令牌的时候，一次性颁发两个令牌，一个用于获取数据，另一个用于获取新的令牌（refresh token 字段）。令牌到期前，用户使用 refresh token 发一个请求，去更新令牌。

>     https://b.com/oauth/token?
>       grant_type=refresh_token&
>       client_id=CLIENT_ID&
>       client_secret=CLIENT_SECRET&
>       refresh_token=REFRESH_TOKEN

上面 URL 中，`grant_type`参数为`refresh_token`表示要求更新令牌，`client_id`参数和`client_secret`参数用于确认身份，`refresh_token`参数就是用于更新令牌的令牌。

B 网站验证通过以后，就会颁发新的令牌。

写到这里，颁发令牌的四种方式就介绍完了

## GitHub OAuth 第三方登录示例教程

很多网站登录时，允许使用第三方网站的身份，这称为"第三方登录"。

![](../images/network/auth/bg2019042101.jpg)

下面就以 GitHub 为例，写一个最简单的应用，演示第三方登录。

### 一、第三方登录的原理

所谓第三方登录，实质就是 OAuth 授权。用户想要登录 A 网站，A 网站让用户提供第三方网站的数据，证明自己的身份。获取第三方网站的身份数据，就需要 OAuth 授权。

举例来说，A 网站允许 GitHub 登录，背后就是下面的流程。

> 1.  A 网站让用户跳转到 GitHub。
> 2.  GitHub 要求用户登录，然后询问"A 网站要求获得 xx 权限，你是否同意？"
> 3.  用户同意，GitHub 就会重定向回 A 网站，同时发回一个授权码。
> 4.  A 网站使用授权码，向 GitHub 请求令牌。
> 5.  GitHub 返回令牌.
> 6.  A 网站使用令牌，向 GitHub 请求用户数据。

下面就是这个流程的代码实现。

### 二、应用登记

一个应用要求 OAuth 授权，必须先到对方网站登记，让对方知道是谁在请求。

所以，你要先去 GitHub 登记一下。当然，我已经登记过了，你使用我的登记信息也可以，但为了完整走一遍流程，还是建议大家自己登记。这是免费的。

访问这个[网址](https://github.com/settings/applications/new)，填写登记表。

![](../images/network/auth/bg2019042102.jpg)

应用的名称随便填，主页 URL 填写`http://localhost:8080`，跳转网址填写 `http://localhost:8080/oauth/redirect`。

提交表单以后，GitHub 应该会返回客户端 ID（client ID）和客户端密钥（client secret），这就是应用的身份识别码。

### 三、示例仓库

我写了一个[代码仓库](https://github.com/ruanyf/node-oauth-demo)，请将它克隆到本地。

>     $ git clone git@github.com:ruanyf/node-oauth-demo.git
>     $ cd node-oauth-demo

两个配置项要改一下，写入上一步的身份识别码。

> -   [`index.js`](https://github.com/ruanyf/node-oauth-demo/blob/master/index.js#L3)：改掉变量`clientID` and `clientSecret`
> -   [`public/index.html`](https://github.com/ruanyf/node-oauth-demo/blob/master/public/index.html#L16)：改掉变量`client_id`

然后，安装依赖。

>     $ npm install

启动服务。

>     $ node index.js

浏览器访问`http://localhost:8080`，就可以看到这个示例了。

### 四、浏览器跳转 GitHub

示例的首页很简单，就是一个链接，让用户跳转到 GitHub。

![](../images/network/auth/bg2019042103.jpg)

跳转的 URL 如下。

>     https://github.com/login/oauth/authorize?
>       client_id=7e015d8ce32370079895&
>       redirect_uri=http://localhost:8080/oauth/redirect

这个 URL 指向 GitHub 的 OAuth 授权网址，带有两个参数：`client_id`告诉 GitHub 谁在请求，`redirect_uri`是稍后跳转回来的网址。

用户点击到了 GitHub，GitHub 会要求用户登录，确保是本人在操作。

### 五、授权码

登录后，GitHub 询问用户，该应用正在请求数据，你是否同意授权。

![](../images/network/auth/bg2019042104.png)

用户同意授权， GitHub 就会跳转到`redirect_uri`指定的跳转网址，并且带上授权码，跳转回来的 URL 就是下面的样子。

>     http://localhost:8080/oauth/redirect?
>       code=859310e7cecc9196f4af

后端收到这个请求以后，就拿到了授权码（`code`参数）。

### 六、后端实现

示例的[后端](https://github.com/ruanyf/node-oauth-demo/blob/master/index.js)采用 Koa 框架编写，具体语法请看[教程](https://www.ruanyifeng.com/blog/2017/08/koa.html)。

这里的关键是针对`/oauth/redirect`的请求，编写一个[路由](https://github.com/ruanyf/node-oauth-demo/blob/master/index.js#L16)，完成 OAuth 认证。

>     const oauth = async ctx => {
>       // ...
>     };
>
>     app.use(route.get('/oauth/redirect', oauth));

上面代码中，`oauth`函数就是路由的处理函数。下面的代码都写在这个函数里面。

路由函数的第一件事，是从 URL 取出授权码。

>     const requestToken = ctx.request.query.code;

### 七、令牌

后端使用这个授权码，向 GitHub 请求令牌。

>     const tokenResponse = await axios({
>       method: 'post',
>       url: 'https://github.com/login/oauth/access_token?' +
>         `client_id=${clientID}&` +
>         `client_secret=${clientSecret}&` +
>         `code=${requestToken}`,
>       headers: {
>         accept: 'application/json'
>       }
>     });

上面代码中，GitHub 的令牌接口`https://github.com/login/oauth/access_token`需要提供三个参数。

> -   `client_id`：客户端的 ID
> -   `client_secret`：客户端的密钥
> -   `code`：授权码

作为回应，GitHub 会返回一段 JSON 数据，里面包含了令牌`accessToken`。

>     const accessToken = tokenResponse.data.access_token;

### 八、API 数据

有了令牌以后，就可以向 API 请求数据了。

>     const result = await axios({
>       method: 'get',
>       url: `https://api.github.com/user`,
>       headers: {
>         accept: 'application/json',
>         Authorization: `token ${accessToken}`
>       }
>     });

上面代码中，GitHub API 的地址是`https://api.github.com/user`，请求的时候必须在 HTTP 头信息里面带上令牌`Authorization: token 361507da`。

然后，就可以拿到用户数据，得到用户的身份。

>     const name = result.data.name;
>     ctx.response.redirect(`/welcome.html?name=${name}`);