# Postman的其他使用

1. 抓包及拦截器

    点击雷达后进入设置页面，设置端口号，然后自定义过滤条件和请求保存位置，接着配置客户端手动代理，即可抓包或者过滤。
2. Postman脚本

    - Pre-request Script 是在请求发送到服务端之前，会运行一次，这里能完成环境变量的设置或者发送一个异步请求。

        比如：预处理中设置环境变量之后，在请求头中可以直接使用环境变量
        ```
        # 设置环境变量
        pm.environment.set("header_timestamp",new Date());
        ```
    - Tests Script 是在获取到响应之后，对请求结果的断言或者再次发送请求等操作。

        比如：使用 pm 对象，通过对响应 Body 的判断，来判断接口返回的数据是否合理。
        ```
        pm.test("Test Result：",function(){
        var jsonData = pm.response.json();

        //直接判断json里面的数据
        //排名第一国家是日本
        pm.expect(jsonData.result[0].country).to.eql("日本");});
        ```
3. 发布接口文档

    选中请求集合，点击 Publish Docs，跳转到集合发布页面，执行发布操作。
    
    最后，将生成一份完善在线的 API 文档，可以分享出去，其他人也可以通过 Postman 导入进行编辑完善。