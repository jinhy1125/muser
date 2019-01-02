# Dom4j解析XML丢失文本中换行符的问题解决

## 问题描述

程序员还是看代码好了：

```java
public static void main(String[] args) throws DocumentException {
    String text = "<root value=\"123\n321\"></root>";
    Document document = DocumentHelper.parseText(text);
    System.out.println(document.valueOf("//root/@value"));
}
```

虽然我的内容里面有换行符，但是打印出的结果是`123 321`。

## 排查过程

最开始怀疑是`document.valueOf()`的问题，然而看document的结构发现里面的属性已经是错误的了。那问题出在`DocumentHelper.parseText()`这里。

然后就是常规跟代码，看看到底是哪里把我们的代码处理成空格的。

最后发现了`com.sun.org.apache.xerces.internal.impl.XMLScanner#scanAttributeValue`,涉及到我们问题的代码片段如下：

```java
else if (c == '\n' || c == '\r') {
    fEntityScanner.scanChar();
    stringBuffer.append(' ');
    if (entityDepth == fEntityDepth && fNeedNonNormalizedValue) {
        fStringBuffer2.append('\n');
    }
}
```

关键行是`stringBuffer.append(' ')`这里解析到有换行符后，他没有原封不动把换行符加入到结果中，而是加入了空格。

这里还有一个信息就是`entityDepth == fEntityDepth && fNeedNonNormalizedValue`条件，我们可以猜想我们可能是可以通过设置参数来影响解析XML的过程的，使它能原封把换行符放入结果，因为时间关系这里没有继续的往下查代码。

## 解决过程

以前想要修改jar里面的代码，我们都是在我们的源码中创建与原类相同的包路径和java文件。这样我们可以编译构建的过程中用我们的代码来覆盖原有jar包中的class。

但是这次照做之后发现没有替换成功。原因在于，`这次想要覆盖的是java的代码`。我们可以想到java的类肯定会优先加载，或者使用自己的classloader进行加载，这样就把我们的意图给破坏掉了。

但是java也不是没有给我们留后门。这时候就有个JVM的配置参数

```bash
-Djava.endorsed.dirs=I:\xml\
```

配置好了这个，JVM就会去配置的目录中找jar并加载。

我们再需要做的就是把编译后的`XMLScanner.class`打成jar包。

使用命令

```bash
jar -cvf xml.jar *
```

并放到上面配置的目录下就可以啦。

## 补充

如果想把自己的类替换到所有的项目，而不想给每个项目都加虚拟机参数，我们可以把修改的jar放入路径：

```
$JAVA_HOME/jre/lib/endorsed
```

## 后续

1. 进一步看看JVM的加载原理及顺序
2. 研究除了配置endorsed，考虑使用javaagent的方式进行增强
3. 研究是不是有相应配置来解决此问题

