# web安全之XSS漏洞

Cross-Site Scripting（跨站脚本攻击）简称 XSS，是一种代码注入攻击。攻击者通过在目标网站上注入恶意脚本，使之在用户的浏览器上运行。利用这些恶意脚本，攻击者可获取用户的敏感信息如 Cookie、SessionID 等，进而危害数据安全。为了和 CSS 区分，这里把攻击的第一个字母改成了 X，于是叫做 XSS。XSS 的本质是：恶意代码未经过滤，与网站正常的代码混在一起；浏览器无法分辨哪些脚本是可信的，导致恶意脚本被执行。

而由于直接在用户的终端执行，恶意代码能够直接获取用户的信息，或者利用这些信息冒充用户向网站发起攻击者定义的请求。在部分情况下，由于输入的限制，注入的恶意脚本比较短。但可以通过引入外部的脚本，并由浏览器执行，来完成比较复杂的攻击策略。

<!--more-->

## 例子

### QQ 邮箱 m.exmail.qq.com 域名反射型 XSS 漏洞

攻击者发现 `http://m.exmail.qq.com/cgi-bin/login?uin=aaaa&domain=bbbb` 这个 URL 的参数 uin、domain 未经转义直接输出到 HTML 中。

于是攻击者构建出一个 URL，并引导用户去点击： `http://m.exmail.qq.com/cgi-bin/login?uin=aaaa&domain=bbbb%26quot%3B%3Breturn+false%3B%26quot%3B%26lt%3B%2Fscript%26gt%3B%26lt%3Bscript%26gt%3Balert(document.cookie)%26lt%3B%2Fscript%26gt%3B`

用户点击这个 URL 时，服务端取出 URL 参数，拼接到 HTML 响应中：

```
<script>
getTop().location.href="/cgi-bin/loginpage?autologin=n&errtype=1&verify=&clientuin=aaa"+"&t="+"&d=bbbb";return false;</script><script>alert(document.cookie)</script>"+"...
```

浏览器接收到响应后就会执行 `alert(document.cookie)`，攻击者通过 JavaScript 即可窃取当前用户在QQ邮箱域名下的 Cookie ，进而危害数据安全。

## 漏洞原理

XSS，跨站脚本攻击，指的是用户输入的内容，被当做html/javascript代码直接填充到HTML文本里面，并被浏览器直接渲染，作为页面元素展示/执行。若用户输入一些有害JavaScript代码，这些代码会被直接执行,导致不同程度的危害，比如敏感信息泄露，登录仿冒等业务危害。常见的XSS注入方法有：

- 在 HTML 中内嵌的文本中，恶意内容以 script 标签形成注入。
- 在内联的 JavaScript 中，拼接的数据突破了原本的限制（字符串，变量，方法名等）。
- 在标签属性中，恶意内容包含引号，从而突破属性值的限制，注入其他属性或者标签。
- 在标签的 href、src 等属性中，包含 javascript: 等可执行代码。
- 在 onload、onerror、onclick 等事件中，注入不受控制代码。
- 在 style 属性和标签中，包含类似 background-image:url("javascript:..."); 的代码（新版本浏览器已经可以防范）。
- 在 style 属性和标签中，包含类似 expression(...) 的 CSS 表达式代码（新版本浏览器已经可以防范）。

总之，如果开发者没有将用户输入的文本进行合适的过滤，就贸然插入到HTML中，这很容易造成注入漏洞。攻击者可以利用漏洞，构造出恶意的代码指令，进而利用恶意代码危害数据安全。此外不仅仅是业务上的UGC(用户生成内容)可以进行注入，包括 URL上的参数等都可以是攻击的来源。在处理输入时，以下内容都不可信：

- 来自用户的 UGC 信息
- 来自第三方的链接
- URL 参数
- POST 参数
- Referer（可能来自不可信的来源）
- Cookie（可能来自其他子域注入）

XSS有以下子类型:

1. 存储型XSS,这种攻击常见于带有用户保存数据的网站功能，如论坛发帖、商品评论、用户私信等。
    - 攻击者将恶意代码提交到目标网站的数据库中。
    - 用户打开目标网站时，网站服务端将恶意代码从数据库取出，拼接在 HTML 中返回给浏览器。
    - 用户浏览器接收到响应后解析执行，混在其中的恶意代码也被执行。
    - 恶意代码窃取用户数据并发送到攻击者的网站，或者冒充用户的行为，调用目标网站接口执行攻击者指定的操作。
2. 反射型XSS,反射型 XSS 跟存储型 XSS 的区别是：存储型 XSS 的恶意代码存在数据库里，反射型 XSS 的恶意代码存在 URL 里。反射型 XSS 漏洞常见于通过 URL 传递参数的功能，如网站搜索、跳转等。由于需要用户主动打开恶意的 URL 才能生效，攻击者往往会结合多种手段诱导用户点击。POST 的内容也可以触发反射型 XSS，只不过其触发条件比较苛刻（需要构造表单提交页面，并引导用户点击），所以非常少见。
    - 攻击者构造出特殊的 URL，其中包含恶意代码。
    - 用户打开带有恶意代码的 URL 时，网站服务端将恶意代码从 URL 中取出，拼接在 HTML 中返回给浏览器。
    - 用户浏览器接收到响应后解析执行，混在其中的恶意代码也被执行。
    - 恶意代码窃取用户数据并发送到攻击者的网站，或者冒充用户的行为，调用目标网站接口执行攻击者指定的操作。  
3. DOM型XSS,DOM 型 XSS 跟前两种 XSS 的区别：DOM 型 XSS 攻击中，取出和执行恶意代码由浏览器端完成，属于前端 JavaScript 自身的安全漏洞，而其他两种 XSS 都属于服务端的安全漏洞。
    - 攻击者构造出特殊的 URL，其中包含恶意代码。
    - 用户打开带有恶意代码的 URL。
    - 用户浏览器接收到响应后解析执行，前端 JavaScript 取出 URL 中的恶意代码并执行。
    - 恶意代码窃取用户数据并发送到攻击者的网站，或者冒充用户的行为，调用目标网站接口执行攻击者指定的操作。

## 防御策略

XSS 攻击有两大要素：

- 攻击者提交恶意代码。
- 浏览器执行恶意代码。

因此在防御XSS时，我们可以针对这两个方面。

- 防止 HTML 中出现注入。
- 防止 JavaScript 执行时，执行恶意代码。

### 输入过滤

1. 如果前端过滤输入，攻击者实际上可以绕过前端过滤，直接构造请求。
2. 如果后端过滤，对数据进行html转义。但是问题在于转义数据后，在某些场景下展示时可能会不正常。
3. 纯接口(服务器间)进行调用的目前不需要进行XSS配置，只要数据被引入到html页面上的均需要进行XSS防御配置。对所有输出的内容进行统一HTML特殊字符转义。若为json格式输出，则仅对key和value进行转义，不破坏json结构。数据经过安全转义之后，输出到页面上，页面展示乱码，需要前端处理。

对于明确的输入类型，例如数字、URL、电话号码、邮件地址等等内容，进行输入过滤还是必要的。但是对于不明确的场景，输入侧过滤引入很大的不确定性和乱码问题。此时需要前端兜底处理。

### 前端可以做什么？

#### 预防存储型和反射型 XSS 攻击

存储型和反射型 XSS 都是在服务端取出恶意代码后，插入到响应 HTML 里的，攻击者刻意编写的"数据"被内嵌到"代码"中，被浏览器所执行。

预防这两种漏洞，有两种常见做法：

- 改成纯前端渲染，把代码和数据分隔开。纯前端渲染的过程：
    - 浏览器先加载一个静态 HTML，此 HTML 中不包含任何跟业务相关的数据。
    - 然后浏览器执行 HTML 中的 JavaScript。
    - JavaScript 通过 Ajax 加载业务数据，调用 DOM API 更新到页面上。
    - 在很多内部、管理系统中，采用纯前端渲染是非常合适的。但对于性能要求高，或有 SEO 需求的页面，我们仍然要面对拼接 HTML 的问题。
- 对HTML做充分转义。如果拼接 HTML 是必要的，就需要采用合适的转义库，对 HTML 模板各处插入点进行充分的转义。


#### 预防DOM型XSS攻击

DOM 型 XSS 攻击，实际上就是网站前端 JavaScript 代码本身不够严谨，把不可信的数据当作代码执行了。

- 在使用 .innerHTML、.outerHTML、document.write() 时要特别小心，不要把不可信的数据作为 HTML 插到页面上，而应尽量使用 .textContent、.setAttribute() 等。
- 如果用 Vue/React 技术栈，并且不使用 v-html/dangerouslySetInnerHTML 功能，就要在前端 render 阶段避免 innerHTML、outerHTML 的 XSS 隐患。
- DOM 中的内联事件监听器，如 location、onclick、onerror、onload、onmouseover 等，`<a>` 标签的 href 属性，JavaScript 的 eval()、setTimeout()、setInterval() 等，都能把字符串作为代码运行。如果不可信的数据拼接到字符串中传递给这些 API，很容易产生安全隐患，请务必避免。

#### Content Security Policy

严格的 CSP 在 XSS 的防范中可以起到以下的作用：

- 禁止加载外域代码，防止复杂的攻击逻辑。
- 禁止外域提交，网站被攻击后，用户的数据不会泄露到外域。
- 禁止内联脚本执行（规则较严格，目前发现 GitHub 使用）。
- 禁止未授权的脚本执行（新特性，Google Map 移动版在使用）。
- 合理使用上报可以及时发现 XSS，利于尽快修复问题。

#### 其他

- HTTP-only Cookie: 禁止 JavaScript 读取某些敏感 Cookie，攻击者完成 XSS 注入后也无法窃取此 Cookie。
- 验证码：防止脚本冒充用户提交危险操作。
- 对于不受信任的输入，都应该限定一个合理的长度。虽然无法完全防止 XSS 发生，但可以增加 XSS 攻击的难度。

## XSS 的检测

- 使用通用 XSS 攻击字符串手动检测 XSS 漏洞。
    - [Unleashing an Ultimate XSS Polyglot](https://github.com/0xsobky/HackVault/wiki/Unleashing-an-Ultimate-XSS-Polyglot)
- 使用扫描工具自动检测 XSS 漏洞。
    - [Arachni](https://github.com/Arachni/arachni)
    - [Mozilla HTTP Observatory](https://github.com/mozilla/http-observatory/)
    - [w3af](https://github.com/andresriancho/w3af)

## 参考

- [1] [Cross-site scripting](https://en.wikipedia.org/wiki/Cross-site_scripting)
- [2] [XSS (Cross Site Scripting) Prevention Cheat Sheet](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)
- [3] [Use the OWASP Java Encoder-Use-the-OWASP-Java-Encoder](https://github.com/OWASP/owasp-java-encoder/wiki/2)
- [4] [Unleashing an Ultimate XSS Polyglot](https://github.com/0xsobky/HackVault/wiki/Unleashing-an-Ultimate-XSS-Polyglot)
- [5] [XSS game](https://xss-game.appspot.com/level1)

