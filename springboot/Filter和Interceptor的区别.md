### Filter和Interceptor的区别

#### Filter

Filter是源于Servlet规范，Filter的全限类名【javax.servlet.Filter 】

可以看到Filter本身是用在tomcat等Web容器进行Servlet相关处理时使用的工具，并非是Spring原生工具，从这个发现中我们不难揣测在Spring中Filter和Interceptor在职能上如此相近，y