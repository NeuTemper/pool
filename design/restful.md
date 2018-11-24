# REST(基于HTTP实现介绍) 
---
## REST
REST全称是Representational State Transfer，REST是[Roy Thomas Fielding](https://en.wikipedia.org/wiki/Roy_Fielding)在他2000年的博士论文[Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)中提出的。他的设计目的如下
    
    My work is motivated by the desire to understand and evaluate the architectural design of network-based application 
    software through principled use of architectural constraints, thereby obtaining the functional, performance, and 
    social properties desired of an architecture.
    写作目的是想在符合架构原理的前提下，理解和评估以网络为基础的应用软件的架构设计，得到一个功能强、性能好、适宜通信的架构。
- ### REST架构风格的架构约束
    - #### Client-Server
        通信只能由客户端单方面发起，表现为请求-响应的形式
    - #### Stateless    
        通信的会话状态（Session State）应该全部由客户端负责维护
    - #### Cache
        响应内容可以在通信链的某处被缓存，以改善网络效率
    - #### Uniform Interface    
        通信链的组件之间通过统一的接口相互通信，以提高交互的可见性
    - #### Layered System
        通过限制组件的行为（即，每个组件只能“看到”与其交互的紧邻层），将架构分解为若干等级的层
    - #### Code-On-Demand
        支持通过下载并执行一些代码（例如Java Applet、Flash或JavaScript），对客户端的功能进行扩展         
    
REST的通常被译成“表现层状态转化”，听起来比较生涩，要理解REST就要理解Representational State Transfer这个词组的每一个词代表了什么涵义  
- ### Resources(资源)
    REST省略了主语表现层指的是“资源”表现层。所谓"资源"，就是网络上的一个实体，或者说是网络上的一个具体信息。它可以是一段文本、一张图片、一首歌曲、一种服务，是一个具体的存在形式。可以用一个URI指向它，每种资源对应一个特定的URI(资源的唯一标识)。要获取这个资源，访问它的URI就可以，因此URI就成了每一个资源的地址或独一无二的识别符
- ### Representation(表现层)
    表现层指的是资源的表现形式，HTTP的[Content-Type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type)实体头部用于指示资源的MIME类型 [media type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)，常用的type如下
    - text/plain
    - text/html
    - image/jpeg
    - image/png
    - audio/mpeg
    - audio/ogg
    - audio/*
    - video/mp4
    - application/*
    - application/json
    - application/xml
    - application/javascript
    - application/octet-stream
- ### State Transfer(状态转化)
    就是客户端和服务器互动的一个过程，由于HTTP是无状态的，资源状态是维护在服务端的，在互动过程中涉及到数据和状态的变化, 这种变化叫做状态转换。
资源是唯一的，对资源的状态改变使用的HTTP动词的对应语义实现对资源数据的增(PUT)删(DELETE)改(PUT/PATCH)查(GET)
## RESTful
REST是一种软件架构风格，RESTful是遵循REST架构风格的(一种实现)
## 如何设计restful API
- ### 通讯协议使用HTTPs
- ### 合理设计uri

        https://example.com/api/books 获取所有书
        https://example.com/api/books 添加一本书
        https://example.com/api/books/bookId 修改一本书
        https://example.com/api/books/bookId 删除一本书    
    
- ### 合理使用[http动词](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)

    - #### GET
        The HEAD method asks for a response identical to that of a GET request, but without the response body.
    - #### POST
        The POST method is used to submit an entity to the specified resource, often causing a change in state or side effects on the server.
    - #### PUT
        The PUT method replaces all current representations of the target resource with the request payload.
    - #### PATCH
        The PATCH method is used to apply partial modifications to a resource.
    - #### DELETE
        The DELETE method deletes the specified resource
        
        
            GET    https://example.com/api/books 获取所有书
            POST   https://example.com/api/books 添加一本书
            PUT    https://example.com/api/books/bookId 修改一本书
            DELETE https://example.com/api/books/bookId 删除一本书       
    
- ### 向客户端返回[状态码](https://www.restapitutorial.com/httpstatuscodes.html)和提示信息

    - #### 200 OK 
        服务器成功返回用户请求的数据，操作是幂等的
    - #### 201 CREATED 
        用户新建或修改数据成功。
    - #### 204 NO CONTENT 
        用户删除数据成功。
    - #### 400 INVALID REQUEST 
        用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的
    - #### 401 Unauthorized 
        表示用户没有权限（令牌、用户名、密码错误）
    - #### 403 Forbidden 
        表示用户得到授权（与401错误相对），但是访问是被禁止的
    - #### 404 NOT FOUND 
        用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的

## restful设计误区
   - ### URI包含动词
   
   
       https://example.com/appName/getBooks 获取所有书
       https://example.com/appName/addBooks 添加一本书
       https://example.com/appName/updateBooks/:bookId 修改一本书
       https://example.com/appName/deleteBooks/:bookId 删除一本书
   - ### 版本号放在URI中
           https://api.example.com/v1/
            
       github的开发文档吗描述了版本号要放在HTTP头部信息中
       
           curl https://api.github.com/users/technoweenie -I
           HTTP/1.1 200 OK
           X-GitHub-Media-Type: github.v3
           curl https://api.github.com/users/technoweenie -I \
            -H "Accept: application/vnd.github.full+json"
           HTTP/1.1 200 OK
           X-GitHub-Media-Type: github.v3; param=full; format=json
           curl https://api.github.com/users/technoweenie -I \
            -H "Accept: application/vnd.github.v3.full+json"
           HTTP/1.1 200 OK
           X-GitHub-Media-Type: github.v3; param=full; format=json
## restful设计优点
- ### 统一接口
    早期的WEB项目前后端是在一起的，但是近年来移动互联网的发展，各种类型的Client层出不穷，RESTful可以通过一套统一的接口为 Web，iOS和Android提供服务
- ### URL具有很强可读性的
    具有自描述性，易于理解
- ### 如果提供无状态的服务接口
    可提高应用的水平扩展性
- ### 解耦
    使异构系统间的通信变得简单
## 开源框架对REST的支持
- ### SpringMVC  


    ```java  
            @RequestMapping(value = "/getBooks", method = {RequestMethod.GET, RequestMethod.POST})
            
            public enum RequestMethod {
                GET,
                HEAD,
                POST,
                PUT,
                PATCH,
                DELETE,
                OPTIONS,
                TRACE;
                private RequestMethod() {
                }
            }
            
springMVC是提供不同的请求方式的，但是很多时候并没有被使用
- ### Jersey


    ```java  
            @Path("/myResource")
            public class SomeResource {
                @GET
                @Consumes("text/plain")
                @Produces({"application/xml", "application/json"})
                public String doGetAsPlainText() {
                    ...
                }
             
                @GET
                @Produces("text/html")
                public String doGetAsHtml() {
                    ...
                }
            }
                    
[Jersey](https://jersey.github.io/documentation/latest/jaxrs-resources.html#d0e2129)在REST方面支持的更加友好
- @Path("/myResource") 指定访问路径
- @Consumes 注释代表的是一个资源可以接受的 MIME 类型
- @Produces 注释代表的是一个资源可以返回的 MIME 类型
