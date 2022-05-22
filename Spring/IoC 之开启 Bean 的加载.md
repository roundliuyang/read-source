#  IoC 之开启 Bean 的加载



![Spring IoC 作用](IoC 之开启 Bean 的加载.assets/181705b2eb48364c7626b59397e2af1c)

​																					*Spring IoC 作用*

(此图参考《Spring 揭秘》)

Spring IoC 容器所起的作用如上图所示，它会以某种方式加载 Configuration Metadata，将其解析注册到容器内部，然后回根据这些信息绑定整个系统的对象，最终组装成一个可用的基于轻量级容器的应用系统。

Spring 在实现上述功能中，将整个流程分为两个阶段：容器初始化阶段和加载bean 阶段。分别如下

**容器初始化阶段**：

