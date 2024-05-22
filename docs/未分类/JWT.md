## [什么是 JWT?](#什么是-jwt)
JWT （JSON Web Token） 是目前最流行的跨域认证解决方案，是一种基于 Token 的认证授权机制。 从 JWT 的全称可以看出，JWT 本身也是 Token，一种规范化之后的 JSON 结构的 Token。
JWT 自身包含了身份验证所需要的所有信息，因此，我们的服务器不需要存储 Session 信息。这显然增加了系统的可用性和伸缩性，大大减轻了服务端的压力。
可以看出，**JWT 更符合设计 RESTful API 时的「Stateless（无状态）」原则** 。
并且， 使用 JWT 认证可以有效避免 CSRF 攻击，因为 JWT 一般是存在在 localStorage 中，使用 JWT 进行身份验证的过程中是不会涉及到 Cookie 的。
我在 [JWT 优缺点分析](/system-design/security/advantages-and-disadvantages-of-jwt.html)这篇文章中有详细介绍到使用 JWT 做身份认证的优势和劣势。
下面是 [RFC 7519open in new window](https://tools.ietf.org/html/rfc7519) 对 JWT 做的较为正式的定义。
JSON Web Token (JWT) is a compact, URL-safe means of representing claims to be transferred between two parties. The claims in a JWT are encoded as a JSON object that is used as the payload of a JSON Web Signature (JWS) structure or as the plaintext of a JSON Web Encryption (JWE) structure, enabling the claims to be digitally signed or integrity protected with a Message Authentication Code (MAC) and/or encrypted. ——[JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
## [JWT 由哪些部分组成？](#jwt-由哪些部分组成)
![](https://raw.githubusercontent.com/danmuking/image/main/b05424eb09c35b8d6a8ece7d23f01164.png)
此图片来源于：[https://supertokens.com/blog/oauth-vs-jwtopen in new window](https://supertokens.com/blog/oauth-vs-jwt)
JWT 本质上就是一组字串，通过（.）切分成三个为 Base64 编码的部分：

- **Header** : 描述 JWT 的元数据，定义了生成签名的算法以及 Token 的类型。
- **Payload** : 用来存放实际需要传递的数据
- **Signature（签名）**：服务器通过 Payload、Header 和一个密钥(Secret)使用 Header 里面指定的签名算法（默认是 HMAC SHA256）生成。

JWT 通常是这样的：xxxxx.yyyyy.zzzzz。
示例：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```
你可以在 [jwt.ioopen in new window](https://jwt.io/) 这个网站上对其 JWT 进行解码，解码之后得到的就是 Header、Payload、Signature 这三部分。
Header 和 Payload 都是 JSON 格式的数据，Signature 由 Payload、Header 和 Secret(密钥)通过特定的计算公式和加密算法得到。
![](https://raw.githubusercontent.com/danmuking/image/main/67c1e556d05d1a1015d976705f55cebd.png)
### [Header](#header)
Header 通常由两部分组成：

- typ（Type）：令牌类型，也就是 JWT。
- alg（Algorithm）：签名算法，比如 HS256。

示例：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
JSON 形式的 Header 被转换成 Base64 编码，成为 JWT 的第一部分。
### [Payload](#payload)
Payload 也是 JSON 格式数据，其中包含了 Claims(声明，包含 JWT 的相关信息)。
Claims 分为三种类型：

- **Registered Claims（注册声明）**：预定义的一些声明，建议使用，但不是强制性的。
- **Public Claims（公有声明）**：JWT 签发方可以自定义的声明，但是为了避免冲突，应该在 [IANA JSON Web Token Registryopen in new window](https://www.iana.org/assignments/jwt/jwt.xhtml) 中定义它们。
- **Private Claims（私有声明）**：JWT 签发方因为项目需要而自定义的声明，更符合实际项目场景使用。

下面是一些常见的注册声明：

- iss（issuer）：JWT 签发方。
- iat（issued at time）：JWT 签发时间。
- sub（subject）：JWT 主题。
- aud（audience）：JWT 接收方。
- exp（expiration time）：JWT 的过期时间。
- nbf（not before time）：JWT 生效时间，早于该定义的时间的 JWT 不能被接受处理。
- jti（JWT ID）：JWT 唯一标识。

示例：

```json
{
  "uid": "ff1212f5-d8d1-4496-bf41-d2dda73de19a",
  "sub": "1234567890",
  "name": "John Doe",
  "exp": 15323232,
  "iat": 1516239022,
  "scope": ["admin", "user"]
}
```
Payload 部分默认是不加密的，**一定不要将隐私信息存放在 Payload 当中！！！**
JSON 形式的 Payload 被转换成 Base64 编码，成为 JWT 的第二部分。
### [Signature](#signature)
Signature 部分是对前两部分的签名，作用是防止 JWT（主要是 payload） 被篡改。
这个签名的生成需要用到：

- Header + Payload。
- 存放在服务端的密钥(一定不要泄露出去)。
- 签名算法。

签名的计算公式如下：

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```
算出签名以后，把 Header、Payload、Signature 三个部分拼成一个字符串，每个部分之间用"点"（.）分隔，这个字符串就是 JWT 。
## [如何基于 JWT 进行身份验证？](#如何基于-jwt-进行身份验证)
在基于 JWT 进行身份验证的的应用程序中，服务器通过 Payload、Header 和 Secret(密钥)创建 JWT 并将 JWT 发送给客户端。客户端接收到 JWT 之后，会将其保存在 Cookie 或者 localStorage 里面，以后客户端发出的所有请求都会携带这个令牌。
 JWT 身份验证示意图
简化后的步骤如下：

1. 用户向服务器发送用户名、密码以及验证码用于登陆系统。
2. 如果用户用户名、密码以及验证码校验正确的话，服务端会返回已经签名的 Token，也就是 JWT。
3. 用户以后每次向后端发请求都在 Header 中带上这个 JWT 。
4. 服务端检查 JWT 并从中获取用户相关信息。

两点建议：

1. 建议将 JWT 存放在 localStorage 中，放在 Cookie 中会有 CSRF 风险。
2. 请求服务端并携带 JWT 的常见做法是将其放在 HTTP Header 的 Authorization 字段中（Authorization: Bearer Token）。

[spring-security-jwt-guideopen in new window](https://github.com/Snailclimb/spring-security-jwt-guide) 就是一个基于 JWT 来做身份认证的简单案例，感兴趣的可以看看。
## [如何防止 JWT 被篡改？](#如何防止-jwt-被篡改)
有了签名之后，即使 JWT 被泄露或者截获，黑客也没办法同时篡改 Signature、Header、Payload。
这是为什么呢？因为服务端拿到 JWT 之后，会解析出其中包含的 Header、Payload 以及 Signature 。服务端会根据 Header、Payload、密钥再次生成一个 Signature。拿新生成的 Signature 和 JWT 中的 Signature 作对比，如果一样就说明 Header 和 Payload 没有被修改。
不过，如果服务端的秘钥也被泄露的话，黑客就可以同时篡改 Signature、Header、Payload 了。黑客直接修改了 Header 和 Payload 之后，再重新生成一个 Signature 就可以了。
**密钥一定保管好，一定不要泄露出去。JWT 安全的核心在于签名，签名安全的核心在密钥。**
## [JWT 的优势](#jwt-的优势)
相比于 Session 认证的方式来说，使用 JWT 进行身份认证主要有下面 4 个优势。
### [无状态](#无状态)
JWT 自身包含了身份验证所需要的所有信息，因此，我们的服务器不需要存储 Session 信息。这显然增加了系统的可用性和伸缩性，大大减轻了服务端的压力。
不过，也正是由于 JWT 的无状态，也导致了它最大的缺点：**不可控！**
就比如说，我们想要在 JWT 有效期内废弃一个 JWT 或者更改它的权限的话，并不会立即生效，通常需要等到有效期过后才可以。再比如说，当用户 Logout 的话，JWT 也还有效。除非，我们在后端增加额外的处理逻辑比如将失效的 JWT 存储起来，后端先验证 JWT 是否有效再进行处理。具体的解决办法，我们会在后面的内容中详细介绍到，这里只是简单提一下。
### [有效避免了 CSRF 攻击](#有效避免了-csrf-攻击)
**CSRF（Cross Site Request Forgery）** 一般被翻译为 **跨站请求伪造**，属于网络攻击领域范围。相比于 SQL 脚本注入、XSS 等安全攻击方式，CSRF 的知名度并没有它们高。但是，它的确是我们开发系统时必须要考虑的安全隐患。就连业内技术标杆 Google 的产品 Gmail 也曾在 2007 年的时候爆出过 CSRF 漏洞，这给 Gmail 的用户造成了很大的损失。
**那么究竟什么是跨站请求伪造呢？** 简单来说就是用你的身份去做一些不好的事情（发送一些对你不友好的请求比如恶意转账）。
举个简单的例子：小壮登录了某网上银行，他来到了网上银行的帖子区，看到一个帖子下面有一个链接写着“科学理财，年盈利率过万”，小壮好奇的点开了这个链接，结果发现自己的账户少了 10000 元。这是这么回事呢？原来黑客在链接中藏了一个请求，这个请求直接利用小壮的身份给银行发送了一个转账请求，也就是通过你的 Cookie 向银行发出请求。
```html
<a src="http://www.mybank.com/Transfer?bankId=11&money=10000"
  >科学理财，年盈利率过万</a
>
```
CSRF 攻击需要依赖 Cookie ，Session 认证中 Cookie 中的 SessionID 是由浏览器发送到服务端的，只要发出请求，Cookie 就会被携带。借助这个特性，即使黑客无法获取你的 SessionID，只要让你误点攻击链接，就可以达到攻击效果。
另外，并不是必须点击链接才可以达到攻击效果，很多时候，只要你打开了某个页面，CSRF 攻击就会发生。
```html
<img src="http://www.mybank.com/Transfer?bankId=11&money=10000" />
```
**那为什么 JWT 不会存在这种问题呢？**
一般情况下我们使用 JWT 的话，在我们登录成功获得 JWT 之后，一般会选择存放在 localStorage 中。前端的每一个请求后续都会附带上这个 JWT，整个过程压根不会涉及到 Cookie。因此，即使你点击了非法链接发送了请求到服务端，这个非法请求也是不会携带 JWT 的，所以这个请求将是非法的。
总结来说就一句话：**使用 JWT 进行身份验证不需要依赖 Cookie ，因此可以避免 CSRF 攻击。**
不过，这样也会存在 XSS 攻击的风险。为了避免 XSS 攻击，你可以选择将 JWT 存储在标记为httpOnly 的 Cookie 中。但是，这样又导致了你必须自己提供 CSRF 保护，因此，实际项目中我们通常也不会这么做。
常见的避免 XSS 攻击的方式是过滤掉请求中存在 XSS 攻击风险的可疑字符串
在 Spring 项目中，我们一般是通过创建 XSS 过滤器来实现的。

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class XSSFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
      FilterChain chain) throws IOException, ServletException {
        XSSRequestWrapper wrappedRequest =
          new XSSRequestWrapper((HttpServletRequest) request);
        chain.doFilter(wrappedRequest, response);
    }

    // other methods
}

```
### [适合移动端应用](#适合移动端应用)
使用 Session 进行身份认证的话，需要保存一份信息在服务器端，而且这种方式会依赖到 Cookie（需要 Cookie 保存 SessionId），所以不适合移动端。
但是，使用 JWT 进行身份认证就不会存在这种问题，因为只要 JWT 可以被客户端存储就能够使用，而且 JWT 还可以跨语言使用。
### [单点登录友好](#单点登录友好)
使用 Session 进行身份认证的话，实现单点登录，需要我们把用户的 Session 信息保存在一台电脑上，并且还会遇到常见的 Cookie 跨域的问题。但是，使用 JWT 进行认证的话， JWT 被保存在客户端，不会存在这些问题。
## 参考资料
[JWT 基础概念详解](https://javaguide.cn/system-design/security/jwt-intro.html#%E5%A6%82%E4%BD%95%E5%9F%BA%E4%BA%8E-jwt-%E8%BF%9B%E8%A1%8C%E8%BA%AB%E4%BB%BD%E9%AA%8C%E8%AF%81)
