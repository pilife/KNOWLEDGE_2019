# [JAVA]注释(元注释和自定义注释)

### 注释的意义//todo
### Java标准注释//todo
Annotations是J2SE 5.0 (Tiger)带来的新特性
### 自定义注释：
#### 定义语法：
- 定义注释
```java
package com.oreilly.tiger.ch06;
/**
 * Marker annotation to indicate that a method or class
 * is still in progress.
 */
public @interface InProgress { }
```
- 添加成员
  当该注释类型的成员变量不是一个的时候，将一个变量命名为 value **没有任何意义**。只要成员变量多于一个，就应该尽可能**准确地为其命名**。
```java
package com.oreilly.tiger.ch06;
/**
 * Annotation type to indicate a task still needs to be
 * completed.
 */
public @interface TODO {
 String value();
}
```
- 成员带默认值
```java
package com.oreilly.tiger.ch06;
public @interface GroupTODO {
 public enum Severity { CRITICAL, IMPORTANT, TRIVIAL, DOCUMENTATION };
 Severity severity() default Severity.IMPORTANT;
 String item();
 String assignedTo();
 String dateAssigned();
}
```

#### 使用语法：
- 使用注释
```java
@com.oreilly.tiger.ch06.InProgress
public void calculateInterest(float amount, float rate) {
 // Need to finish this method later
}
```
- 使用带成员的注释：简写形式
```java
@com.oreilly.tiger.ch06.InProgress
@TODO("Figure out the amount of interest per month")
public void calculateInterest(float amount, float rate) {
 // Need to finish this method later
}
```
- 使用带成员的注释：加长版形式
```java
 @com.oreilly.tiger.ch06.InProgress
 @GroupTODO(
 item="Figure out the amount of interest per month",
 assignedTo="Brett McLaughlin",
 dateAssigned="08/04/2004"
 )
 public void calculateInterest(float amount, float rate) {
 // Need to finish this method later
 }
```
### 元注释: 注释的注释（annotating annotations）
@Target( The ElementType enum)
@Retention( The RetentionPolicy enum)
@Documented: 
	- Marker annotation have no member variables.
	- require specifies the annotation's retention as RUNTIME.
@Inherited:当被注释的类的子类也需要自动继承这个标记的时候（默认是不继承的）

### Conclusion
现在，您也许已经准备回到 Java 世界为所有的事物编写文档和注释了。但是要注意的是：
- 文档是用来帮助理清容易混淆的类和方法的。而不是滥用（比如getXXX,setXXX，根本不会有人去看）
- 注释可能会出现同样的趋势，尽管程度较低。 经常使用**标准注释**类型是一个好主意，甚至很重要。 每个Java 5编译器都会支持它们，并且它们的行为已被很好地理解。 但是，如果要使用定制注释和元注释，那么就很难保证花费很大力气创建的那些类型在您的开发环境之外还有什么意 义。因此要慎重。在合理的情况下使用注释，不要荒谬使用。无论如何，注释都是一种很好的工具，可以在开发过程中提供真正的帮助。

结论原文如下：
At this point, you're ready to go back into the Java world and document and annotate everything.Then again, this reminds me a bit of what happened when everyone figured out Javadoc. We all
went into the mode of over-documenting everything, before someone realized that Javadoc is best used for clarification of confusing classes or methods. Nobody looks at those easy-to-understand getXXX() and setXXX() methods you worked so hard to Javadoc.
The same trend will probably occur with annotations, albeit to a lesser degree. It's a great idea to use the standard annotation types often, and even heavily. Every Java 5 compiler will support them, and their behavior is well-understood. However, as you get into custom annotations and
meta-annotations, it becomes harder to ensure that the types you work so hard to create have any meaning outside of your own development context. So be measured. Use annotations when it makes sense to, but don't get ridiculous. However you use it, an annotation facility is nice to have and can really help out in your development process.


Reference:
[Add metadata to Java](https://www.ibm.com/developerworks/library/j-annotate1/j-annotate1-pdf.pdf)
[Custom annotations](https://www.ibm.com/developerworks/library/j-annotate2/j-annotate2-pdf.pdf)