# springmvc
在post请求中
@RequestParam是用来处理 Content-Type 为 application/x-www-form-urlencoded

@RequestBody 是用来处理 Content-Type 为 application/json

前端请求

1. Request Payload  
    ```
     Content-Type:application/json
       {"name":"niba"}
    ```
2.  Form Data
    ```
    Content-Type ：application/x-www-form-urlencoded
    name=niba
    ```
后台处理
1.  Request Payload 请求， 
    必须加 @RequestBody 才能将请求正文解析到对应的 bean 中,且只能通过 request.getReader() 来获取请求正文内容
2. Form Data
    无需任何注解，springmvc 会自动使用 MessageConverter 将请求参数解析到对应的 bean，且通过 request.getParameter(...) 能获取请求参数