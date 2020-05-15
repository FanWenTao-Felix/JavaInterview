### spring boot特性，优势，适用场景等

> - 使用JavaConfig有助于避免使用XML
> - 避免大量的maven导入和各种版本冲突
> - 通过提供默认值快速开始开发
> - 没有单独的web服务器需要，这就意味着不再需要启动Tomcat、Glassfish或其他任何东西
> - 需要更少的配置，因为没有web.xml文件。只需添加用@Configuration注释的类，然后添加用@Bean注释的方法，Spring将自动加载对象并像以前一样对其进行管理。

### SpringBoot starter工作原理：

> 1、SpringBoot在启动时扫描项目依赖的jar包，寻找包含spring.factories文件的jar
>
> 2、根据spring.factories配置加载AutoConfigure
>
> 3、根据@Conditional注解的条件，进行自动配置并将bean注入到Spring Context