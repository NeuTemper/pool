# 文档规范

## 如何添加文档
添加文档，请遵守规范，未遵守文档，不允许合并分支。

### 标题

具体可以参阅[中文技术文档的写作规范-标题][1]

### 文本
具体可以参阅[中文技术文档的写作规范-文本][2]
文本里面有几个地方请注意一下，我们写文档的时候，尽量如此，从阅览的角度上，更舒服。

##### 英文处理
正确的方式应该是，在中文添加`English`的时候，把听英文做标注处理。这里更加适合于我们写文档的时候，文档中有一些专业名词。

##### 代码块
在添加代码块的时候，请同时添加代码语言。如：
```java
public class Singleton {  
    private static class SingletonHolder {  
        private static final Singleton uniqueInstance = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
        return SingletonHolder.uniqueInstance;
    }  
}
```

### 参考文献
有任何参考文献，请在文章末尾注明参考文献。

例如：
- [参考文献-博客名][1]
- [参考文献-博客名][2]




[1]:[https://github.com/ruanyf/document-style-guide/blob/master/docs/title.md]
[2]:[https://github.com/ruanyf/document-style-guide/blob/master/docs/text.md]
