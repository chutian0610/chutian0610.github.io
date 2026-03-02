# web安全之CSRF

CSRF即跨站点请求伪造(Cross—Site Request Forgery), 在CSRF攻击中,攻击者盗用了你的身份，以你的名义发送恶意请求，对服务器来说这个请求是完全合法的，但是却完成了攻击者所期望的一个操作，比如以你的名义发送邮件、发消息，盗取你的账号，添加系统管理员，甚至于购买商品、虚拟货币转账等。

<!--more-->

## 攻击原理

假设Web A为存在CSRF漏洞的网站，Web B为攻击者构建的恶意网站，User C为Web A网站的合法用户。

1. 用户C打开浏览器，访问受信任网站A，输入用户名和密码请求登录网站A;
2. 在用户信息通过验证后，网站A产生Cookie信息并返回给浏览器，此时用户登录网站A成功，可以正常发送请求到网站A;
3. 用户未退出网站A之前，在同一浏览器中，打开一个TAB页访问网站B;
4. 网站B接收到用户请求后，返回一些攻击性代码，并发出一个请求要求访问第三方站点A;
5. 浏览器在接收到这些攻击性代码后，根据网站B的请求，在用户不知情的情况下携带Cookie信息，向网站A发出请求。网站A并不知道该请求其实是由B发起的，所以会根据用户C的Cookie信息以C的权限处理该请求，导致来自网站B的恶意代码被执行。

> 数据库/持久层内容增删改的业务接口若没有进行实现签名机制/token随机化，原理上均存在CSRF漏洞。

## 例子

以删除用户差评为例，假设有一个接口是: `http://www.testA.com/deleteComment?id=3322`。假设攻击者A是黑灰产，负责删除用户差评，此时他得知用户B对某个业务范围内的商品做了差评，评论id为322134。此时他构造`http://www.testA.com/deleteComment?id=322134`.并且将该url转换为短链接:`http://www.testB.com/xx/yy`。接下来他私聊用户，告知用户中了10w大奖，领奖链接为:`http://www.testB.com/xx/yy`。当用户点击后差评即被删除(用户刚刚登录过,session未过期)。

实际攻击中短链接可能是隐藏在页面中的img标签的src,也不仅是GET请求,也可能是隐藏的POST请求。其实可以看出，CSRF攻击是源于WEB的隐式身份验证机制！WEB的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的！

## 防御策略

### http refer 字段

根据http协议，在http头中有一个字段叫 Referer，它记录了该 HTTP 请求的来源地址。要防御 CSRF 攻击，只需要针对每一个可以改变数据的服务请求，验证其 Referer 值，如果是以正确的域名开头，则说明该请求是合法的。如果 Referer 是其他网站的话，则有可能是黑客的 CSRF 攻击，拒绝该请求。

这种方法的显而易见的好处就是简单易行，网站的普通开发人员不需要操心 CSRF 的漏洞，只需要在最后给所有安全敏感的请求统一增加一个拦截器来检查 Referer 的值就可以。特别是对于当前现有的系统，不需要改变当前系统的任何已有代码和逻辑，没有风险，非常便捷。

但是这种方法并非万无一失。Referer 的值是由浏览器提供的，虽然 HTTP 协议上有明确的要求，但是每个浏览器对于 Referer 的具体实现可能有差别，并不能保证浏览器自身没有安全漏洞。使用验证 Referer 值的方法，就是把安全性都依赖于第三方（即浏览器）来保障，从理论上来讲，这样并不安全。事实上，对于某些浏览器，比如 IE6 或 FF2，目前已经有一些方法可以篡改 Referer 值。即便是使用最新的浏览器，黑客无法篡改 Referer 值，这种方法仍然有问题。因为 Referer 值会记录下用户的访问来源，有些用户认为这样会侵犯到他们自己的隐私权，特别是有些组织担心 Referer 值会把组织内网中的某些信息泄露到外网中。因此，用户自己可以设置浏览器使其在发送请求时不再提供 Referer。当他们正常访问银行网站时，网站会因为请求没有 Referer 值而认为是 CSRF 攻击，拒绝合法用户的访问。

### 请求中添加随机token

CSRF 攻击之所以能够成功，是因为黑客可以完全伪造用户的请求，该请求中所有的用户验证信息都是存在于 cookie 中，因此黑客可以在不知道这些验证信息的情况下直接利用用户自己的 cookie 来通过安全验证。要抵御 CSRF，关键在于在请求中放入黑客所不能伪造的信息。可以在 HTTP 请求中以参数的形式加入一个随机产生的 token，并在服务器端建立一个拦截器来验证这个 token，如果请求中没有 token 或者 token 内容不正确，则认为可能是 CSRF 攻击而拒绝该请求。

这种方法要比检查 Referer 要安全一些，token 可以在用户登陆后产生并放于 session 之中，然后在每次请求时把 token 从 session 中拿出，与请求中的 token 进行比对，但这种方法的难点在于如何把 token 以参数的形式加入请求。

1. 对于使用后端模板渲染的页面，可以在form表单中添加隐藏输入框`<input type=”hidden” name=”csrftoken” value=”tokenvalue”/>`,这样在提交时就会把token以参数形式加入请求。
2. ajax发起请求的情况下，token无法直接渲染到页面上，可以在cookie中读取token，将其带入到ajax请求的参数中。然后传到后端。JavaScript读取cookie中的Csrf_token这步。由于浏览器的同源策略，攻击者是无法从被攻击者的cookie中读取任何东西的。所以，攻击者无法成功发起CSRF攻击。
3. 在一般业务场景下，安全包会将token种到服务端对应的域名cookie下，可以被前端js调用和植入到header或者参数中。但是在跨域场景下，前端页面与后端服务端不是同一个域名，导致无法取到服务端域名下的cookie。假设aaaa.com要跨域访问bbbb.com的接口，bbbb.com的接口做了csrf校验(注意跨域的处理)。
  1. bbbb.com新增一个接口，返回自身的csrf token.
  2. aaaa.com携带with-credentials的头部来获取该token.
  3. 将获取到的token存放在客户端上(比如localstorage，或者页面隐藏字段中)
  4. aaaa.com 访问bbbb.com其他接口的时候，获取token作为接口参数/头部参数传递给bbbb.com，同时访问该接口时应该设置withcredentials = true:

跨域接口例子:

```java
@RequestMapping(value = "/ajax", produces = MediaType.APPLICATION_JSON_VALUE)
@CrossOrigin(origins = "http://aaaaaa.com:7001", maxAge = 3600)
@ResponseBody
public CsrfToken getCsrfToken(HttpServletRequest request, HttpServletResponse response) {
    CsrfToken csrfToken = csrfTokenRepository.loadToken(request);
    if (csrfToken == null) {
        csrfToken = csrfTokenRepository.generateToken(request);
        csrfTokenRepository.saveToken(csrfToken, request, response);
    }
    return csrfToken;
}
```

### 验证码

每次的用户提交都需要用户在表单中填写一个图片上的随机字符串，但可能会降低用户体验。

