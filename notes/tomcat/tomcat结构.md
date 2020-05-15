### tomcat结构，类加载器流程

> ![image-20200105154748245](./../image/image-20200105154748245.png)
>
> 目录结构： 
> • /bin：存放windows或Linux平台上启动和关闭Tomcat的脚本文件 
> • /conf：存放Tomcat服务器的各种全局配置文件，其中最重要的是server.xml和web.xml 
> • /doc：存放Tomcat文档 
> • /server：包含三个子目录：classes、lib和webapps 
> • /server/lib：存放Tomcat服务器所需的各种JAR文件 
> • /server/webapps：存放Tomcat自带的两个WEB应用admin应用和 manager应用 
> • /common/lib：存放Tomcat服务器以及所有web应用都可以访问的jar文件 
> • /shared/lib：存放所有web应用都可以访问的jar文件（但是不能被Tomcat服务器访问） 
> • /logs：存放Tomcat执行时的日志文件 
> • /src：存放Tomcat的源代码 
> • /webapps：Tomcat的主要Web发布目录，默认情况下把Web应用文件放于此目录 
> • /work：存放JSP编译后产生的class文件
>
> 
>
> 类加载器模式，双亲委派模式： 
>
> ![image-20200105154853067](./../image/image-20200105154853067.png)