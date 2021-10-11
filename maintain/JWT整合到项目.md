## SpringBoot中使用 JWT + 拦截器实现登录验证。

### 旧的方法存在缺点

之前的策略是，UUID + redis + 拦截器的思路。

服务器端在验证 roomid 和 password相匹配之后，使用  UUID 生成一个字符串作为 token ，接着往 Redis 服务中写入一个映射(token, roomid)， 设置过期时间为20分钟， 并且把token 通过响应返回给客户端。因此，客户端便可以使用这个 UUID 产生的 token 来访问。

对于需要携带 token 的请求，安排了一个拦截器来检查这些请求，检验请求所携带的 token 与 roomid 是否与 Redis 中已有的映射相匹配，若匹配，说明已经授权可以可以放行，若不匹配，则拒绝请求。

缺点在于：

- 服务器端需要使用 Redis 保存许多键值对，占用一定的内存空间。
- 客户端获取的值是单纯的值，当用户太多时，UUID 的数目变得过多会加大被猜中的概率。

### JWT 

token 的三部分中，第一部分是头，第二部分是负载，第三部分是签名，一二部分都是base64加密，不能存放敏感信息。
如果客户端私自修改过期时间，token 的验证环节可以检测出来。（有使用密钥对过期时间进行加密）
如果修改客户端 roomid，是否意味着可以冒用他人的 token？token 生成过程有把负载部分考虑在内，验证的时候可以验证出负载被修改。
使用 JWT 是用时间换空间，节省了服务器端的空间。

### 使用 JWT（JSON Web Token） 的方法

maven 中添加依赖，引入 Java-jwt 项目。

```java
<dependency>
        <groupId>com.auth0</groupId>
        <artifactId>java-jwt</artifactId>
        <version>3.18.2</version>
</dependency>
```

服务器有一个**不能公开**的密钥。

服务器端在验证 roomid 和 password 之后，Token 签名算法使用**密钥**根据 payload 的内容生成签名，可以配置token 的过期时间（本项目配置的是 1 小时），token 的生成相当于服务器对验证通过者的授权，**授权其可进行 1 小时的免密码畅行**。最终得到一个 token 通过响应返回给客户端。 客户端可以使用这个 token 进行请求。

对于需要携带 token 的请求，安排了一个拦截器进行检查，使用 token 校验算法来检查请求所携带的 token的签名是否有效（过期或者 roomid 与签名内容不匹配都会被认为无效）。若 token 检验正确，则放行，若检验错误，则拒绝请求。

下方是 token 的生成与验证，其中 secret 是自己设定的密钥（是一个字符串，我的密钥可不能公开贴出来）。

token 生成过程：

```java
 public static String generate(String roomid){
        String token = "";
        try{
            Algorithm algorithm = Algorithm.HMAC256(secret);
            Map<String,Object> map = new HashMap<>();
            map.put("roomid",roomid);
            token = JWT.create().withIssuer("auth0")
                    .withPayload(map)
                    .withExpiresAt(new Date(System.currentTimeMillis() + (long)3600*1000))
                    .sign(algorithm);
        }catch(JWTCreationException e){
            e.printStackTrace();
        }
        return token;
    }
```

token 的验证，验证通过则返回 roomid，验证不通过则返回 null。

```java
public static String validate(String roomid, String token){
        DecodedJWT jwt = null;
        try{
            Algorithm algorithm = Algorithm.HMAC256(secret);
            JWTVerifier jwtVerifier = JWT.require(algorithm)
                    .withClaim("roomid",roomid)
                    .withIssuer("auth0").build();
            jwt = jwtVerifier.verify(token);
            if (jwt == null )return null;
            Claim claim = jwt.getClaim("roomid");
            if (claim == null )return null;
            return claim.asString();
        }catch (JWTVerificationException e){
            return null;
        }

    }
```

参考代码：https://github.com/auth0/java-jwt

