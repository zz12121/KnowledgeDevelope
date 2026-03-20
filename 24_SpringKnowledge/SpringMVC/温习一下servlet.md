# Servlet核心知识

## 概述

Servlet是Java Web应用的核心组件，运行在Web容器中，用于处理客户端的HTTP请求并返回响应。

## 生命周期

Servlet生命周期包含三个阶段：

### 1. 初始化（init）

```java
public void init(ServletConfig config) throws ServletException {
    // 初始化资源，只调用一次
}
```

- 第一次访问Servlet时调用
- 可在web.xml或注解中配置load-on-startup

### 2. 服务（service）

```java
public void service(ServletRequest req, ServletResponse res) 
    throws ServletException, IOException {
    // 根据请求方法分发到doGet/doPost
}
```

- 每次请求都会调用
- 根据HTTP方法分发到doGet、doPost等

### 3. 销毁（destroy）

```java
public void destroy() {
    // 释放资源，只调用一次
}
```

- Web应用停止时调用
- 用于释放数据库连接等资源

## 工作原理

1. 客户端发送HTTP请求
2. Web容器创建Request和Response对象
3. 容器根据URL映射找到对应Servlet
4. 创建或复用Servlet实例
5. 调用service()方法
6. 容器将响应返回客户端

## 核心接口

### HttpServlet

最常用的Servlet实现类：

```java
public class MyServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
            throws ServletException, IOException {
        // 处理GET请求
    }
    
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) 
            throws ServletException, IOException {
        // 处理POST请求
    }
}
```

## 请求对象（HttpServletRequest）

```java
// 获取请求参数
String name = request.getParameter("name");

// 获取请求头
String contentType = request.getContentType();

// 获取Session
HttpSession session = request.getSession();

// 获取Cookie
Cookie[] cookies = request.getCookies();

// 获取请求路径
String uri = request.getRequestURI();
```

## 响应对象（HttpServletResponse）

```java
// 设置响应内容
response.setContentType("text/html");
response.setCharacterEncoding("UTF-8");

// 获取输出流
PrintWriter out = response.getWriter();
out.write("Hello");

// 设置响应头
response.setHeader("Cache-Control", "no-cache");

// 重定向
response.sendRedirect("/anotherPage");
```

## ServletContext

Web应用上下文，整个应用共享：

```java
ServletContext context = getServletContext();
context.setAttribute("key", "value");
```

## 面试常见问题

1. **Servlet是线程安全的吗？**
   - 不是，Servlet是单实例多线程
   - 不要在Servlet中定义共享变量

2. **Servlet与CGI的区别？**
   - CGI：每个请求创建新进程
   - Servlet：多线程处理请求

3. **load-on-startup的作用？**
   - 指定Servlet启动顺序
   - 正数越小越先启动

## 总结

Servlet是Java Web开发的基础，理解其生命周期和工作原理对于掌握Spring MVC等框架至关重要。
