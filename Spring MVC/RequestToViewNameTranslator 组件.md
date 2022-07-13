# RequestToViewNameTranslator 组件



## 1. 概述

本文，我们来分享 RequestToViewNameTranslator 组件。在 [《精尽 Spring MVC 源码分析 —— 组件一览》](http://svip.iocoder.cn/Spring-MVC/Components-intro) 中，我们对它已经做了介绍：

`org.springframework.web.servlet.RequestToViewNameTranslator` ，请求到视图名的转换器接口。代码如下：

```
// RequestToViewNameTranslator.java

public interface RequestToViewNameTranslator {

	/**
     * 根据请求，获得其视图名
	 */
	@Nullable
	String getViewName(HttpServletRequest request) throws Exception;

}
```

- 在 DispatcherServlet 中，我们已经看到，在 ModelAndView 不存在对应的视图时，会通过 RequestToViewNameTranslator 来获取默认的视图名，作为其视图。

## 2. DefaultRequestToViewNameTranslator

`org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator` ，实现 RequestToViewNameTranslator 接口，默认且是唯一的 RequestToViewNameTranslator 实现类。

### 2.1 构造方法

```
// DefaultRequestToViewNameTranslator.java

private static final String SLASH = "/";

/**
 * 前缀
 */
private String prefix = "";
/**
 * 后缀
 */
private String suffix = "";

/**
 * 分隔符
 */
private String separator = SLASH;

/**
 * 是否移除开头 {@link #SLASH}
 */
private boolean stripLeadingSlash = true;
/**
 * 是否移除末尾 {@link #SLASH}
 */
private boolean stripTrailingSlash = true;
/**
 * 是否移除拓展名
 */
private boolean stripExtension = true;

/**
 * URL 路径工具类
 */
private UrlPathHelper urlPathHelper = new UrlPathHelper();
```

- 不要看变量这么多，实际灰常简单。一起继续往下瞅瞅。

### 2.2 getViewName

实现 `#getViewName(HttpServletRequest request)` 方法，代码如下：

```
// DefaultRequestToViewNameTranslator.java

@Override
public String getViewName(HttpServletRequest request) {
    // 获得请求路径
    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
    // 获得视图名
    return (this.prefix + transformPath(lookupPath) + this.suffix);
}
```

- `<1>` 处，获得请求路径。

- `<2>` 处，调用 `#transformPath(String lookupPath)` 方法，转换请求路径，后续在拼接上 `prefix` 和 `suffix` ，形成最终的视图名。代码如下：

  ```
  // DefaultRequestToViewNameTranslator.java
  
  @Nullable
  protected String transformPath(String lookupPath) {
      String path = lookupPath;
      // 移除开头 SLASH
      if (this.stripLeadingSlash && path.startsWith(SLASH)) {
          path = path.substring(1);
      }
      // 移除末尾 SLASH
      if (this.stripTrailingSlash && path.endsWith(SLASH)) {
          path = path.substring(0, path.length() - 1);
      }
      // 移除拓展名
      if (this.stripExtension) {
          path = StringUtils.stripFilenameExtension(path);
      }
      // 替换分隔符
      if (!SLASH.equals(this.separator)) {
          path = StringUtils.replace(path, SLASH, this.separator);
      }
      return path;
  }
  ```

  - 好咧，这就是这个类的核心。😈 没有其它代码了。哈哈哈哈。

## 666. 彩蛋

开心么？？？哈哈哈哈？？？？

参考和推荐如下文章：

- 韩路彪 [《看透 Spring MVC：源代码分析与实践》](https://item.jd.com/11807414.html) 的 [「第15章 RequestToViewNameTranslator」](http://svip.iocoder.cn/Spring-MVC/RequestToViewNameTranslator/#) 小节