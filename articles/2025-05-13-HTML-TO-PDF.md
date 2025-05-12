# 用 iText 将 HTML 生成 PDF

证券公司清算之后会为客户生成包含上一个交易日资产/交易记录的结单 PDF，常用 Thymeleaf 搭配 iText 完成，Thymeleaf 用于将动态数据渲染到 HTML 上，iText 则把 HTML 转为 PDF。

## 引入 POM

```
<!-- iText9 -->
<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>itext-core</artifactId>
  <version>9.1.0</version>
  <type>pom</type>
</dependency>

<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>bouncy-castle-adapter</artifactId>
  <version>9.1.0</version>
</dependency>

<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>html2pdf</artifactId>
  <version>6.1.0</version>
</dependency>

<!-- 数据渲染引擎 -->
<dependency>
  <groupId>org.thymeleaf</groupId>
  <artifactId>thymeleaf</artifactId>
  <version>3.1.3.RELEASE</version>
</dependency>
```

## 渲染动态数据

准备 HTML 样式，可以在 iText 官方提供的工具[页面](https://itextpdf.com/demos/convert-html-css-to-pdf-free-online)调试，完成之后，在 HTML 文件里写入模版语言，例如数据的遍历：

```html
<tbody th:if="${not #lists.isEmpty(asset.assetItems)}">
  <tr th:each="assetItem : ${asset.assetItems}">
    <td th:text="${assetItem.currency}">THB</td>
    <td th:text="${assetItem.balance}">9,802,779.88</td>
    <td th:text="${assetItem.balance}">9,802,779.88</td>
  </tr>

<tr>
  <td colspan="2" class="total-text">TOTAL VALUE</td>
  <td class="total-value" th:text="${asset.totalBalance}">100,000</td>
</tr>
</tbody>
```

准备数据，并放入一个 map 中：

```java
...
Map<String, Object> variables = new HashMap<>();
variables.put("userInfo", userInfo);
variables.put("asset", asset);
variables.put("trade", trade);

```

将 HTML 转为 PDF：

```java
// 加载特定字体，而不是跟随系统
private byte[] loadFont() throws IOException {
    try (InputStream inputStream = Thread.currentThread().getContextClassLoader()
         .getResourceAsStream("templates/font.ttf")) {
      ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
      byte[] buffer = new byte[1024];
      int bytesRead;
      while ((bytesRead = inputStream.read(buffer)) != -1) {
        byteArrayOutputStream.write(buffer, 0, bytesRead);
      }
      return byteArrayOutputStream.toByteArray();
    }
}

public byte[] renderHTML2PDF(String templateName, Map<String, Object> variables) throws IOException {
  	// 加载特定字体
    ConverterProperties properties = new ConverterProperties();
    FontSet fontSet = new FontSet();
    fontSet.addFont(loadFont());
    FontProvider fontProvider = new FontProvider(fontSet);
    properties.setFontProvider(fontProvider);

    // 设置 HTML 模版路径，例如 templates/statement.html
    ClassLoaderTemplateResolver resolver = new ClassLoaderTemplateResolver();
    resolver.setPrefix("templates/");
    resolver.setSuffix(".html");
    resolver.setTemplateMode(TemplateMode.HTML);
    resolver.setCharacterEncoding("UTF-8");
    resolver.setCacheable(false);

    // 设置动态参数
    Context context = new Context();
    context.setVariables(variables);

    // 渲染
    TemplateEngine templateEngine = new TemplateEngine();
    templateEngine.setTemplateResolver(resolver);
    String renderedHtml = templateEngine.process(templateName, context);

    // 输出为 byte[]，方便写入云存储
    try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()){
    	HtmlConverter.convertToPdf(renderedHtml, outputStream, properties);
    	return outputStream.toByteArray();
    }
  	// 或者输出到本地文件
    try (PdfWriter writer = new PdfWriter("local file path");){
      HtmlConverter.convertToPdf(renderedHtml, writer, properties);
    }
}
```

可能遇到的问题：

1. 非英文无法渲染，**需要找到对应的字体库**；
2. 通过 Thymeleaf 渲染之后的 HTML 与渲染前的样式有出入，有些样式并不支持，需要换种样式实现；

参考：

[iText](https://itextpdf.com/demos/convert-html-css-to-pdf-free-online)

[Thymeleaf](https://www.thymeleaf.org/)

