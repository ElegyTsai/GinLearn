# Gin框架学习
## Go module
Go mod 是Go语言用于进行包管理的一个工具，类似于java的maven或者python的pip.使用统一环境(gopath)已经是过去时了，现在都是用go mod来对包环境进行一个管理。在GoLand里在创建项目时会自动创建go.mod来实现包的管理   
在项目的文件夹下使用go get就可以安装指定的包，可以在go.mod中看到变化   
例如对于Gin框架   
```
go get -u github.com/gin-gonic/gin
```
就可以使用import对包进行导入
```go
import(
	"github.com/gin-gonic/gin"
	
	"net/http"
)
```
## Gin 路由
gin框架路由基于httprouter
起主要结构是用
```go
r := gin.Default()//生成一个默认路由
r.Get("/",func(c *gin.Context){
	c.String(http.StatusOK,"hello world")
})
// r.http请求方式(url,func）这样的格式来暴露出url

r.Run(":8080")
//8080表示监听端口
```
- 注意用Restful风格的API
- 可以通过Context的Param来获取url里的参数
```go
//在url里可以获取指定的参数
r.Get("/:name/*action",func(c *gin.Context){
	//用 c.Param来获取/Wei/doit
	name := c.Param("name")
	action := c.Param("action")
	
})
```
- 通过DefaultQuery()或者Query()来获取get的参数
```go
r.Get("/",func(c *gin.Context){
	//用 c.DefaultQuery
	name := c.DefaultQuery("name","DefaultName")
	//或者用Query，如果不存在就返回一个空串
	name := c.Query("name")
	
})
```
Context内部类似是一个编码的map，用指定方法来获取参数
- 对于Post提交的表单，可以用PostForm()方法来获取
```go
r.POST("/form", func(c *gin.Context) {
        types := c.DefaultPostForm("type", "post")
        username := c.PostForm("username")//
```
- 上传文件的方法用Context的FormFile方法实现
```go    
    r.MaxMultipartMemory = 8 << 20
    r.POST("/upload", func(c *gin.Context) {
	    file, err := c.FormFile("file")
	    /* 如果是多个文件的话
	    form, err := c.MultipartForm()
	    files := form.File["files"]
	          // 遍历所有图片
	        for _, file := range files {}
	    */
	if err != nil {
		c.String(500, "上传图片出错")
	}
	// c.JSON(200, gin.H{"message": file.Header.Context})
	c.SaveUploadedFile(file, file.Filename)
	c.String(http.StatusOK, file.Filename)
```
- routes group可以用来管理相同的url
```go
   r := gin.Default()
   // 路由组1 ，处理GET请求
   v1 := r.Group("/v1")
   // {} 是书写规范
   {
      v1.GET("/login", login)
      v1.GET("submit", submit)
   }
```
- 文件比较大了之后需要分割成不同文件来实现路由

##  数据解析
### Json数据解析
```go
   r.POST("loginJSON", func(c *gin.Context) {
      // 声明接收的变量
      var json Login
      // 将request的body中的数据，自动按照json格式解析到结构体
      if err := c.ShouldBindJSON(&json); err != nil {
         // 返回错误信息
         // gin.H封装了生成json数据的工具
         c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
         return
      }
```
### 表单数据解析
也是Bind方法来做
```go
    r.POST("/loginForm", func(c *gin.Context) {
        // 声明接收的变量
        var form Login
        // Bind()默认解析并绑定form格式
        // 根据请求头中content-type自动推断
        if err := c.Bind(&form); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
```
### URL数据解析
用ShouldBindUri方法来做
```go
    r.GET("/:user/:password", func(c *gin.Context) {
        // 声明接收的变量
        var login Login
        // Bind()默认解析并绑定form格式
        // 根据请求头中content-type自动推断
        if err := c.ShouldBindUri(&login); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
```
**总结都是用自定义的结构体，再使用特定的bind方法来自动绑定**

## 渲染
- 对于返回提供了包括xml,yaml,json甚至是ProtoBuf这些数据格式
- 也可以用html模板+数据的方式返回
- 重定向用 ` c.Redirect(http.StatusMovedPermanently, "http://www.123.com")`
- **在GO里可以用Goroutine机制很好的实现异步处理，但是需要注意用一个只读副本，不用原始上下文**
```go
    r.GET("/long_async", func(c *gin.Context) {
        // 需要搞一个副本
        copyContext := c.Copy()
        // 异步处理
        go func() {
            time.Sleep(3 * time.Second)
            log.Println("异步执行：" + copyContext.Request.URL.Path)
        }()
```
## 中间件
- 全局中间件用 use的方式实现
```go
func MiddleWare() gin.HandlerFunc {
    return func (c *gin.Context) {
    }
}
    }
    r := gin.Default()
    // 注册中间件
    r.Use(MiddleWare())
```
- next()方式通常是指完成了响应后再去执行一些步骤，比如统计运行时间** 那中间件本身有顺序吗？**
- 局部中间件
```go
    //局部中间键使用,在get方法中声明要使用的中间件 
    r := gin.Default()
	r.GET("/ce", MiddleWare(), func(c *gin.Context) {
        })
```
- 有非常多已有的中间件，不需要重复造轮子[中间件推荐](https://www.topgoer.com/gin%E6%A1%86%E6%9E%B6/gin%E4%B8%AD%E9%97%B4%E4%BB%B6/%E4%B8%AD%E9%97%B4%E4%BB%B6%E6%8E%A8%E8%8D%90.html)
## 会话控制
获取cookie和设置cookie
```go
cookie, err := c.Cookie("key_cookie")
      if err != nil {
         cookie = "NotSet"
         // 给客户端设置cookie
         //  maxAge int, 单位为秒
         // path,cookie所在目录
         // domain string,域名
         //   secure 是否智能通过https访问
         // httpOnly bool  是否允许别人通过js获取自己的cookie
         c.SetCookie("key_cookie", "value_cookie", 60, "/",
            "localhost", false, true)
      }
```
通常可以用一个自定义的中间件来实现cookie的校验
cookie的缺点有
- 不安全，明文 **（是不是可以通过一个附带的校验码比如单向加密，来保证cookie不被修改，再用中间件来进行验证，）**
- 增加带宽消耗（不能什么东西都往cookie里放，影响性能）
- 可以被禁用
- cookie有上限


Sessions是将数据存储在后段，然后通过一个key来验证。   
cookie数据在用户端，sessions数据在服务端。但是sessionId通常也可能存在cookie里面
```go
var store = sessions.NewCookieStore([]byte("something-very-secret"))
```
## 参数验证
- Gin框架的结构体数据验证可以不用解析数据，比较简洁
```go
//Person ..
type Person struct {
    //不能为空并且大于10
    Age      int       `form:"age" binding:"required,gt=10"`
    Name     string    `form:"name" binding:"required"`
    Birthday time.Time `form:"birthday" time_format:"2006-01-02" time_utc:"1"`
}
```
- 也可以自己写if else 来实现验证

## 日志
可以用这样的方式把日志和控制台一起输出，MultiWriter可以接受多个参数
```go
    f, _ := os.Create("gin.log")
    gin.DefaultWriter = io.MultiWriter(f,os.Stdout)
```

## 热加载
这个功能类似于Spring的热修改，提高开发效率
安装
```bash
    go get -u github.com/cosmtrek/air
```
配置文件应该[看这里](https://www.topgoer.com/gin%E6%A1%86%E6%9E%B6/%E5%85%B6%E4%BB%96/Air%E5%AE%9E%E6%97%B6%E5%8A%A0%E8%BD%BD.html)

## 验证码轮子
captcha是一个验证码的类库[原博](https://www.topgoer.com/gin%E6%A1%86%E6%9E%B6/%E5%85%B6%E4%BB%96/gin%E9%AA%8C%E8%AF%81%E7%A0%81.html)

## token验证
jwt.go里封装了很多这样用于加密的轮子

