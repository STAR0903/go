# Gin-jwt

jwt-go 库的基本使⽤详⻅链接：[QAQ](jwt-go 库的基本使⽤详⻅我的博客链接：https://www.liwenzhou.com/posts/Go/jwt_in_gin/)

#### 鉴权中间件开发

```
const (
 ContextUserIDKey = "userID"
)
var (
 ErrorUserNotLogin = errors.New("当前⽤户未登录")
)
// JWTAuthMiddleware 基于JWT的认证中间件
func JWTAuthMiddleware() func(c *gin.Context) {
 return func(c *gin.Context) {
 // 客户端携带Token有三种⽅式 1.放在请求头 2.放在请求体 3.放在URI
 // 这⾥假设Token放在Header的中
 // 这⾥的具体实现⽅式要依据你的实际业务情况决定
 authHeader := c.Request.Header.Get("Auth")
 if authHeader == "" {
 ResponseErrorWithMsg(c, CodeInvalidToken, "请求头缺少Auth Token")
 c.Abort()
 return
 }
 mc, err := utils.ParseToken(authHeader)
 if err != nil {
 ResponseError(c, CodeInvalidToken)
 c.Abort()
 return
 }
 // 将当前请求的username信息保存到请求的上下⽂c上
 c.Set(ContextUserIDKey, mc.UserID)
 c.Next() // 后续的处理函数可以⽤过c.Get("userID")来获取当前请求的⽤户信息
 }
}
```

#### ⽣成access token和refresh token

```
// GenToken ⽣成access token 和 refresh token
func GenToken(userID int64) (aToken, rToken string, err error) {
 // 创建⼀个我们⾃⼰的声明
 c := MyClaims{
 userID, // ⾃定义字段
 jwt.StandardClaims{
 ExpiresAt: time.Now().Add(TokenExpireDuration).Unix(), // 过期时间
 Issuer: "bluebell", // 签发⼈
 },
 }
 // 加密并获得完整的编码后的字符串token
 aToken, err = jwt.NewWithClaims(jwt.SigningMethodHS256,
c).SignedString(mySecret)
 // refresh token 不需要存任何⾃定义数据
 rToken, err = jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.StandardClaims{
 ExpiresAt: time.Now().Add(time.Second * 30).Unix(), // 过期时间
 Issuer: "bluebell", // 签发⼈
 }).SignedString(mySecret)
 // 使⽤指定的secret签名并获得完整的编码后的字符串token
 return
 }
```

#### 解析access token

```
// ParseToken 解析JWT
func ParseToken(tokenString string) (claims *MyClaims, err error) {
 // 解析token
 var token *jwt.Token
 claims = new(MyClaims)
 token, err = jwt.ParseWithClaims(tokenString, claims, keyFunc)
 if err != nil {
 return
 }
 if !token.Valid { // 校验token
 err = errors.New("invalid token")
 }
 return
}
```

#### refresh token

```
// RefreshToken 刷新AccessToken
func RefreshToken(aToken, rToken string) (newAToken, newRToken string, err
error) {
 // refresh token⽆效直接返回
 if _, err = jwt.Parse(rToken, keyFunc); err != nil {
 return
 }
 // 从旧access token中解析出claims数据
 var claims MyClaims
 _, err = jwt.ParseWithClaims(aToken, &claims, keyFunc)
 v, _ := err.(*jwt.ValidationError)
 // 当access token是过期错误 并且 refresh token没有过期时就创建⼀个新的access token
 if v.Errors == jwt.ValidationErrorExpired {
 return GenToken(claims.UserID)
 }
 return
}
```
