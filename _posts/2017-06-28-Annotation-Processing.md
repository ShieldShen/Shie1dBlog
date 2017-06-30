---
title:        "注解处理器"
description:  "注解是Java 5就出来的一个javac工具，被广泛运用在很多框架与项目上，本文是一篇翻译学习博客"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2017-06-28"
---
# 注解处理器
## 写在前面的
为啥会写这篇博客呢，因为别人都用啊~~ 当然，正经说，注解处理器是运用在编译时期，它现在被大家发掘的用法是来生成一些文件，无论是java还是xml还是简单的键值对文档。那么这个的作用就很明显了，它可以帮我们来生成一些固定的东西，例如重复的代码逻辑，重复的配置文件。这么抽象？不不不，你看看`ButterKnife`、`Dagger`，虽然最后生成出来的代码可能比你一行一行硬写的要多不少，但是嘛，程序员时间才是最宝贵的。老写一样的代码这种体力劳动，嘿嘿...

这篇文章嘛主要是大致翻译的[ANNOTATION PROCESSING101](http://hannesdorfmann.com/annotation-processing/annotationprocessing101)，也有其他博主前辈翻译过啊（他们是正儿八经翻译，我是纯扯淡...）戳链接[Java注解处理器](https://www.race604.com/annotation-processing/)，他们好像还有什么授权问题，然而我是来误人子弟的，也不盈利，**侵删**~~
### 关于注解 - Annotation（作为年轻人这些东西我都是听说的）
注解最早出现在`Java 5`（没有考证哇，人家这么说的，我也无所谓它什么时候出现的╮(╯_╰)╭）。注解是一种用来`描述数据的数据`，在Java中就是用来描述字段函数类等等的。为什么要有注解这个东西呢，据说是因为当时没有注解的时候，大家都用xml来描述数据，但是呢，一些人觉得xml太特么没法看了，就有了注解（依然不可考，毕竟我是年轻人╮(╯_╰)╭）。而注解出来了以后呢，伟大的程序员们拿这个做了些什么东西呢。
* 生成文档，javadoc
* 编译器的检查：`@Override` `@Deprecated` `@SuppressWarnings`
* 编译时相关处理：这篇文章的主要目的，在编译时对代码做一些处理。
* 运行时相关处理：通常使用反射来使用，既可以通过注解来找目标，也可以通过目标来获取注解内容

由于博主很懒，注解的相关基础介绍只能送你个电梯了->[Java Annotation Tutorials](https://docs.oracle.com/javase/tutorial/java/annotations/),另外`Java 8`对于注解还是出了一些新的特性，比如重复注解还有类型注解，还有一些新的预设注解，所以这个教程还是值得看一看的，反正很短~
## 一些小小的预备知识，不想看就跳过，反正你最后会回来看
### APT（ Android Annotation Processing Tool）
首先是电梯直达[android-apt Bitbucket](https://bitbucket.org/hvisser/android-apt)

这个插件是用来辅助`Annotation Processor`在Android Studio上的工作的，让项目在仅在编译时可以使用Annotation Processor的功能来生成文件，并且生成的文件可以在工程中得到正确的引用。
### JavaPoet
一个用来生成java文件的工具，它是Square公司的一个开源项目，按照惯例，送你电梯[JavaPoet in GitHub](https://github.com/square/javapoet/graphs/contributors)

JavaPoet用来写java源文件不要太easy....
## 开始正文内容(不正经的翻译)
### 写在前面的
首先这篇博文讲的不是运行时的注解，而是编译时注解。

注解处理器是一个用来在编译时扫描处理注解的javac工具。我们可以给指定的注解注册相应的注解处理器。在注解处理器里面，你可以根据注册的相应的注解来做相应的处理，我们这里会去生成一些`.java`文件，并使用这些生成的文件实现一个工厂方法。
### AbstractProcessor
`AbstractProcessor`是实现注解处理器必须要继承的类，一般来说需要实现里面的四个方法
```java : n
package com.example;

public class MyProcessor extends AbstractProcessor {

	@Override
	public synchronized void init(ProcessingEnvironment env){ }

	@Override
	public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

	@Override
	public Set<String> getSupportedAnnotationTypes() { }

	@Override
	public SourceVersion getSupportedSourceVersion() { }

}
```
* init(ProcessingEnvironment env):每个处理器**必须有一个空的构造方法**，`init()`方法会由Annotation Processing Tool 调用，并传入一个`ProcessingEnviroment`对象作为参数，从这个对象里面我们可以获得`Elements`，`Types`,`Filer`这三个辅助类的对象，这三个对象的作用会在后面介绍。
* process(Set<? extends TypeElement> annotations, RoundEnvironment env):这个函数是处理器的主要函数，我们可以在这个函数里面进行对特定的注解进行相应处理。其中`RoundEnvironment`可以用来查询被特定注解标记的元素。
* getSupportedAnnotationTypes():这个方法是用来指定这个处理器处理哪些注解。
* getSupportedSourceVersion(): 用来指定使用的Java版本，一般来说返回`SourceVersion.latestSupported()`就可以。

在`Java 7`以上版本里面，我们可以使用注解来代替`getSupportedAnnotationTypes()`,`getSupportedSourceVersion()`：

```java : n
@SupportedSourceVersion(SourceVersion.latestSupported())
@SupportedAnnotationTypes({
   // Set of full qullified annotation type names
 })
public class MyProcessor extends AbstractProcessor {

	@Override
	public synchronized void init(ProcessingEnvironment env){ }

	@Override
	public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }
}
```
然而由于兼容性的问题，特别是Android平台上，还是重写这两个方法比较好。

值得一提的是，javac工具会启动一个独立的jvm来运行注解处理器，也就是说，所有适用于一般项目的lib以及工程模式在注解处理器中统统适用。
### 注册你自己的处理器
如何注册一个处理器到javac中？在最终打包好的处理器jar中，你的工程应该看上去像这样
```
MyProcessor.jar
	- com
		- example
			- MyProcessor.class
            ...
	- META-INF
		- services
			- javax.annotation.processing.Processor
```
也就是要有一个名为`javax.annotation.processing.Processor`的文件放在jar包中`META-INF/services`文件夹下，而这个文件的内容是你这个jar包中所包含的处理器的完全类名，一行一个~，ep：
```
com.example.MyProcessor
com.foo.OtherProcessor
net.blabla.SpecialProcessor
```
这样，javac会从jar包中`javax.annotation.processing.Processor`读取相应的信息，并把这些处理器注册进去。
###一个工厂模式的例子
是时候来一个举个栗子了，然而这里我很不要脸的把栗子换成了自己的，送你电梯[直达我的项目](https://github.com/ShieldShen/AnnotationProcessorDemo)，而且原作者的例子是一个单纯的java工程，我的例子是在Android环境下哦。
当然，这个例子原作者也说了，没什么实际的卵用，而且使用注解处理器的大部分需求不是简单的一两行代码可以实现，这里的例子就是个例子~~~这里会使用处理器来实现一个简单工厂模式的代码生成。
那么，需求来了：一个披萨店，提供两种披萨，披萨A披萨B~~还有一个点心T。
一般来说，我们的代码可以这样写：
```java : n
public interface Meal {
  public float getPrice();
}

public class MargheritaPizza implements Meal {

  @Override public float getPrice() {
    return 6.0f;
  }
}

public class CalzonePizza implements Meal {

  @Override public float getPrice() {
    return 8.5f;
  }
}

public class Tiramisu implements Meal {

  @Override public float getPrice() {
    return 4.5f;
  }
}
```
工厂是这个样子:
```java :n
public class PizzaStore {

  public Meal order(String mealName) {

    if (mealName == null) {
      throw new IllegalArgumentException("Name of the meal is null!");
    }

    if ("Margherita".equals(mealName)) {
      return new MargheritaPizza();
    }

    if ("Calzone".equals(mealName)) {
      return new CalzonePizza();
    }

    if ("Tiramisu".equals(mealName)) {
      return new Tiramisu();
    }

    throw new IllegalArgumentException("Unknown meal '" + mealName + "'");
  }
}
```
就如上面的代码，如果你每添加一种披萨或者点心，都要在`order(String name)`中添加相应的代码。这里这么麻烦，我们是不是可以让代码自己写出来，毕竟都是机械的工作~。那么，下面就来用注解处理器来实现这个功能

### @Factory 注解
首先我们要定义一个注解，用它来标识哪个类被用于工厂模式。
```java : n
@Target(ElementType.TYPE) 
@Retention(RetentionPolicy.CLASS)
public @interface Factory {

  /**
   * The name of the factory
   */
  Class type();

  /**
   * The identifier for determining which item should be instantiated
   */
  String id();
}
```
作用目标是类，编译时可见。`type`表示这个类属于啥，这里就是Meal，`id`就是用于工厂创建时创建这个类的id。这个注解定义好了以后，开始用起来
```java:n
@Factory(
    id = "Margherita",
    type = Meal.class
)
public class MargheritaPizza implements Meal {

  @Override public float getPrice() {
    return 6f;
  }
}
```

```java:n
@Factory(
    id = "Calzone",
    type = Meal.class
)
public class CalzonePizza implements Meal {

  @Override public float getPrice() {
    return 8.5f;
  }
}
```

```java:n
@Factory(
    id = "Tiramisu",
    type = Meal.class
)
public class Tiramisu implements Meal {

  @Override public float getPrice() {
    return 4.5f;
  }
}
```
我们这个`@Factory`的注解虽然定义了在`@Target(ElementType.Type)`上，但是这几个被注解的类是需要在工厂中使用的，那么这些被注解的类就会有一些限制，以保证在`process`过程中不会出现问题：
1. 因为这些被注解的元素需要在工厂类中被实例化，所以只有`类`才可以被`@Factory`注解，而`抽象类`以及`接口`是不允许被`@Factory`注解的。
2. 我们使用工厂方法去创建对象的时候，入参必然是相同的，所以，这些被`@Factory`注解了的类需要有`有相同参数的构造方法`，在我们的例子里面，我们统一使用空参的默认构造方法。
3. 工厂方法创建的对象必然是继承自同一个父类，这个例子里面就是`Meal`。
4. 在被`@Factory`注解的类中`Type`相同的，也就是有同一个父类的，可以在同一个工厂方法中被创建，而我们为这个方法取名，可以使用这个`Type`的值。这个例子中，`type = Meal.class` 最后生成的相应的工厂类的名字就可以是`MealFactory`。
5. `@Factory`中的`id`属性我们这里规定只能是**字符串类型，且必须唯一**。
### 下面就是处理器的写法了
原作者是号称手把手教学，hiahia。

首先，除了最主要的`process()`方法，其他方法大概这个样子：
```java:n
@AutoService(Processor.class)
public class FactoryProcessor extends AbstractProcessor {

  private Types typeUtils;
  private Elements elementUtils;
  private Filer filer;
  private Messager messager;
  private Map<String, FactoryGroupedClasses> factoryClasses = new LinkedHashMap<String, FactoryGroupedClasses>();

  @Override
  public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    typeUtils = processingEnv.getTypeUtils();
    elementUtils = processingEnv.getElementUtils();
    filer = processingEnv.getFiler();
    messager = processingEnv.getMessager();
  }

  @Override
  public Set<String> getSupportedAnnotationTypes() {
    Set<String> annotataions = new LinkedHashSet<String>();
    annotataions.add(Factory.class.getCanonicalName());
    return annotataions;
  }

  @Override
  public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.latestSupported();
  }

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	...
  }
}
```
在处理器类上面有个`@AutoService(Processor.class)`注解，这个注解是用来生成前面说的那个`META-INF/services/javax.annotation.processing.Processor `让我们不需要手动配置，从这里也可以看到，注解处理器也可以使用注解，而javac会启动一个又一个单独的jvm来做这些处理~~

### 介绍一下init(ProcessingEnvironment)方法的操作
在init()中我们获取到了这样几个对象
* **ElementUtils**：帮助处理**Element**的工具类。
* **TypeUtiles**：帮助处理**TypeMirror**的工具类。
* **Filer**：可以用来生成文件。

一个类里面可以分成好几种`Element`。从另一个方面讲，`Element`代表了像`package`,`class`,`method`,`field`等，每个元素代表一个静态的语言级的机构。
``` java :n
package com.example;	// PackageElement

public class Foo {		// TypeElement

	private int a;		// VariableElement
	private Foo other; 	// VariableElement

	public Foo () {} 	// ExecuteableElement

	public void setA ( 	// ExecuteableElement
	                 int newA	// TypeElement
	                 ) {}
}
```
在处理被注解的元素的时候，你要把这些元素看成结构化的文本，而结构化的文本是可以上下检索的。

例如上面的`Foo`类是一个`TypeElement`，我们可以检索它的子元素：
```java:n
TypeElement fooClass = ... ;
for (Element e : fooClass.getEnclosedElements()){ // iterate over children
	Element parent = e.getEnclosingElement();  // parent == fooClass
}
```
如上所述的，  `Elements`其实就是表示的注解作用的代码。`TypeElement`表示的就是代码中`类`。然而，`TypeElement`并不含有这个类确切的信息，在`TypeElement`中你可以获得这个类的名字，但是拿不到诸如父类等信息。但是一个类的父类这些信息，可以通过`TypeMirror`来获取，而获取一个`Element`的`TypeMirror`你可以通过调用`element.asType()`方法获取。
### process方法
#### 获取被@Factory注解了的类
```java:n
  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    // Itearate over all @Factory annotated elements
    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {
  		...
    }
  }
```
这里`RoundEnvironment#getElementsAnnotatedWith(Class)`就是获取一个被相应注解注解了的类元素列表，获取了元素，我们需要按照前面我们设定的一些规定去校验我们的注解是否注解在了正确的元素上。
```java:n
@Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {
      // Check if a class has been annotated with @Factory
      if (annotatedElement.getKind() != ElementKind.CLASS) {
    		...
      }
   }
   ...
}
```
在这里，我们使用了`ElementKind.class`去确认元素是类，而不是使用`annotationElement instanceof TypeElement`，因为接口以及抽象类的元素类型也是TypeElement。
#### 插播一下关于错误处理，也就是日志打印
在`init()`方法中我们获取了一个`Messager`对象。这个对象就是用来在注解处理器中打印各种信息的。其实这个是用来打印一些信息给使用者看得，让他们知道使用错误在哪里，但是这个被用在开发中做debug用也可以。打印的消息有几种，[官方文档](http://docs.oracle.com/javase/7/docs/api/javax/tools/Diagnostic.Kind.html)，其中最重要的是`Kind.ERROR`，因为这种类型的消息可以用来指示我们的注解处理器在哪里失败了。有的时候注解处理器的使用者错误的使用了注解，这个时候如果我们把异常抛出去就会导致运行处理器的jvm终止，并且给使用者丢出一些难以理解的log。总而言之，善于用好Message可以让你的注解处理器的易用性大大提升。
#### 回到process()方法中
使用Messager做错误打印
```java:n
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      // Check if a class has been annotated with @Factory
      if (annotatedElement.getKind() != ElementKind.CLASS) {
        error(annotatedElement, "Only classes can be annotated with @%s",
            Factory.class.getSimpleName());
        return true; // Exit processing
      }
      ...
    }
private void error(Element e, String msg, Object... args) {
    messager.printMessage(
    	Diagnostic.Kind.ERROR,
    	String.format(msg, args),
    	e);
  }
 }
```
为了获取Messager的打印信息，我们必须保证运行处理器的jvm没有崩溃，所以我们在打印了信息之后返回了true。
#### 数据模型
在我们通过注解进行相关操作前，我们得根据之前的规定来对数据进行一些校验。值得一提的是，注解处理器本身就是一个java应用，我们一般用在项目中的编程方法一样可以用在注解处理器程序中。

尽管我们的注解处理器是一个简单的程序，但并不妨碍我们按照面向对象编程思想去编程。我们使用`FactoryAnnotatedClass`这个对象来记录被注解的类的信息。
```java:n
public class FactoryAnnotatedClass {

  private TypeElement annotatedClassElement;
  private String qualifiedSuperClassName;
  private String simpleTypeName;
  private String id;

  public FactoryAnnotatedClass(TypeElement classElement) throws IllegalArgumentException {
    this.annotatedClassElement = classElement;
    Factory annotation = classElement.getAnnotation(Factory.class);
    id = annotation.id();

    if (StringUtils.isEmpty(id)) {
      throw new IllegalArgumentException(
          String.format("id() in @%s for class %s is null or empty! that's not allowed",
              Factory.class.getSimpleName(), classElement.getQualifiedName().toString()));
    }

    // Get the full QualifiedTypeName
    try {
      Class<?> clazz = annotation.type();
      qualifiedSuperClassName = clazz.getCanonicalName();
      simpleTypeName = clazz.getSimpleName();
    } catch (MirroredTypeException mte) {
      DeclaredType classTypeMirror = (DeclaredType) mte.getTypeMirror();
      TypeElement classTypeElement = (TypeElement) classTypeMirror.asElement();
      qualifiedSuperClassName = classTypeElement.getQualifiedName().toString();
      simpleTypeName = classTypeElement.getSimpleName().toString();
    }
  }

  /**
   * Get the id as specified in {@link Factory#id()}.
   * return the id
   */
  public String getId() {
    return id;
  }

  /**
   * Get the full qualified name of the type specified in  {@link Factory#type()}.
   *
   * @return qualified name
   */
  public String getQualifiedFactoryGroupName() {
    return qualifiedSuperClassName;
  }


  /**
   * Get the simple name of the type specified in  {@link Factory#type()}.
   *
   * @return qualified name
   */
  public String getSimpleFactoryGroupName() {
    return simpleTypeName;
  }

  /**
   * The original element that was annotated with @Factory
   */
  public TypeElement getTypeElement() {
    return annotatedClassElement;
  }
}
```
其中，需要着重关注的几行代码:
```java:n
Factory annotation = classElement.getAnnotation(Factory.class);
id = annotation.id(); // Read the id value (like "Calzone" or "Tiramisu")

if (StringUtils.isEmpty(id)) {
    throw new IllegalArgumentException(
          String.format("id() in @%s for class %s is null or empty! that's not allowed",
              Factory.class.getSimpleName(), classElement.getQualifiedName().toString()));
    }
```
在这几行代码里面，我们去获取了`@Factory`注解，并获取了注解的id，校验id是否合法。而这里直接抛出异常的，可以在外部去捕获。这样做的原因有两个：
1. 我们希望像正常应用一样去写这个代码
2. 我们可以在处理器中统一处理错误。

接下来，我们获取了`@Factory`的type字段，并且获取了type的相关信息
```java:n
 try {
      Class<?> clazz = annotation.type();
      qualifiedGroupClassName = clazz.getCanonicalName();
      simpleFactoryGroupName = clazz.getSimpleName();
    } catch (MirroredTypeException mte) {
      DeclaredType classTypeMirror = (DeclaredType) mte.getTypeMirror();
      TypeElement classTypeElement = (TypeElement) classTypeMirror.asElement();
      qualifiedGroupClassName = classTypeElement.getQualifiedName().toString();
      simpleFactoryGroupName = classTypeElement.getSimpleName().toString();
    }
```
在这里有个小的地方需要注意一下，因为type里面存放的是一个java.lang.Class对象，我们在注解处理器在编译源码运行前需要考虑到两点：
1. 这个类已经被编译：这种情况出现在我们包含的是.jar文件中的注解。在这个情况下，我们按照在try代码块中的方式去处理
2. 这个类还没有被编译：在这种情况下，我们的源码还没有编译，而我们直接去获取Class对象是获取不到的，会抛出`MirroredTypeException`。但是在`MirroredTypeException`中，包含了一个TypeMirror对象，从这个对象里面，我们也可以获取相关信息，就像上面catch代码块里的

现在我们再定义一个类`FactoryGroupedClasses`，用这个类去记录所有的`FactoryAnnotatedClass`。因为我们的注解处理器可能会生成好多种工厂类，所以用一个`FactoryGroupedClasses`去记录一个工厂里要生成的类。
```java:n
public class FactoryGroupedClasses {

  private String qualifiedClassName;

  private Map<String, FactoryAnnotatedClass> itemsMap =
      new LinkedHashMap<String, FactoryAnnotatedClass>();

  public FactoryGroupedClasses(String qualifiedClassName) {
    this.qualifiedClassName = qualifiedClassName;
  }

  public void add(FactoryAnnotatedClass toInsert) throws IdAlreadyUsedException {

    FactoryAnnotatedClass existing = itemsMap.get(toInsert.getId());
    if (existing != null) {
      throw new IdAlreadyUsedException(existing);
    }

    itemsMap.put(toInsert.getId(), toInsert);
  }

  public void generateCode(Elements elementUtils, Filer filer) throws IOException {
	...
  }
}
```
因为每个工厂里面的id应该唯一，所以我们用map来存储数据，而generateCode()方法是最后用来生成工厂代码的。

#### 数据校验
根据我们写代码之前的预设规则来校验数据。
```java:n
public class FactoryProcessor extends AbstractProcessor {

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    for (Element annotatedElement : roundEnv.getElementsAnnotatedWith(Factory.class)) {

      ...

      // We can cast it, because we know that it of ElementKind.CLASS
      TypeElement typeElement = (TypeElement) annotatedElement;

      try {
        FactoryAnnotatedClass annotatedClass =
            new FactoryAnnotatedClass(typeElement); // throws IllegalArgumentException

        if (!isValidClass(annotatedClass)) {
          return true; // Error message printed, exit processing
         }
       } catch (IllegalArgumentException e) {
        // @Factory.id() is empty
        error(typeElement, e.getMessage());
        return true;
       }

   	   ...
   }


 private boolean isValidClass(FactoryAnnotatedClass item) {

    // Cast to TypeElement, has more type specific methods
    TypeElement classElement = item.getTypeElement();

    if (!classElement.getModifiers().contains(Modifier.PUBLIC)) {
      error(classElement, "The class %s is not public.",
          classElement.getQualifiedName().toString());
      return false;
    }

    // Check if it's an abstract class
    if (classElement.getModifiers().contains(Modifier.ABSTRACT)) {
      error(classElement, "The class %s is abstract. You can't annotate abstract classes with @%",
          classElement.getQualifiedName().toString(), Factory.class.getSimpleName());
      return false;
    }

    // Check inheritance: Class must be childclass as specified in @Factory.type();
    TypeElement superClassElement =
        elementUtils.getTypeElement(item.getQualifiedFactoryGroupName());
    if (superClassElement.getKind() == ElementKind.INTERFACE) {
      // Check interface implemented
      if (!classElement.getInterfaces().contains(superClassElement.asType())) {
        error(classElement, "The class %s annotated with @%s must implement the interface %s",
            classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
            item.getQualifiedFactoryGroupName());
        return false;
      }
    } else {
      // Check subclassing
      TypeElement currentClass = classElement;
      while (true) {
        TypeMirror superClassType = currentClass.getSuperclass();

        if (superClassType.getKind() == TypeKind.NONE) {
          // Basis class (java.lang.Object) reached, so exit
          error(classElement, "The class %s annotated with @%s must inherit from %s",
              classElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
              item.getQualifiedFactoryGroupName());
          return false;
        }

        if (superClassType.toString().equals(item.getQualifiedFactoryGroupName())) {
          // Required super class found
          break;
        }

        // Moving up in inheritance tree
        currentClass = (TypeElement) typeUtils.asElement(superClassType);
      }
    }

    // Check if an empty public constructor is given
    for (Element enclosed : classElement.getEnclosedElements()) {
      if (enclosed.getKind() == ElementKind.CONSTRUCTOR) {
        ExecutableElement constructorElement = (ExecutableElement) enclosed;
        if (constructorElement.getParameters().size() == 0 && constructorElement.getModifiers()
            .contains(Modifier.PUBLIC)) {
          // Found an empty constructor
          return true;
        }
      }
    }

    // No empty constructor found
    error(classElement, "The class %s must provide an public empty default constructor",
        classElement.getQualifiedName().toString());
    return false;
  }
}
```
这里显然主要的方法是`isValidClass()`:
* 元素必须声明为public ： classElement.getModifiers().contains(Modifier.PUBLIC)
* 元素不能是抽象 ： classElement.getModifiers().contains(Modifier.ABSTRACT)
* 元素必须有与注解中type相同的父类：首先我们从之前取到的type去获取type的TypeMirror，elementUtils.getTypeElement(item.getQualifiedFactoryGroupName())，然后我们去检查它是否是接口，是接口的话，如果是接口的话，我们直接检查元素是否实现了这个接口：classElement.getInterfaces().contains(superClassElement.asType())。否则的话，我们需要循环遍历直到找到对应的类
* 元素必须有public的空参构造方法：classElement.getEnclosedElements()和找到ElementKind.CONSTRUCTOR,然后判断是否是 Modifier.PUBLIC 并且 constructorElement.getParameters().size() == 0
如果这些条件都满足，我们就可以继续往下走，不然打印信息。
#### 给注解的类按照Type分类
在校验数据合格过后，我们需要把生成的`FactoryAnnotatedClass`放到相应的`FactoryGroupedClasses`中：
```java:n
 public class FactoryProcessor extends AbstractProcessor {

   private Map<String, FactoryGroupedClasses> factoryClasses =
      new LinkedHashMap<String, FactoryGroupedClasses>();


 @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

      ...

      try {
        FactoryAnnotatedClass annotatedClass =
            new FactoryAnnotatedClass(typeElement); // throws IllegalArgumentException

          if (!isValidClass(annotatedClass)) {
          return true; // Error message printed, exit processing
        }

        // Everything is fine, so try to add
        FactoryGroupedClasses factoryClass =
            factoryClasses.get(annotatedClass.getQualifiedFactoryGroupName());
        if (factoryClass == null) {
          String qualifiedGroupName = annotatedClass.getQualifiedFactoryGroupName();
          factoryClass = new FactoryGroupedClasses(qualifiedGroupName);
          factoryClasses.put(qualifiedGroupName, factoryClass);
        }

        // Throws IdAlreadyUsedException if id is conflicting with
        // another @Factory annotated class with the same id
        factoryClass.add(annotatedClass);
      } catch (IllegalArgumentException e) {
        // @Factory.id() is empty --> printing error message
        error(typeElement, e.getMessage());
        return true;
      } catch (IdAlreadyUsedException e) {
        FactoryAnnotatedClass existing = e.getExisting();
        // Already existing
        error(annotatedElement,
            "Conflict: The class %s is annotated with @%s with id ='%s' but %s already uses the same id",
            typeElement.getQualifiedName().toString(), Factory.class.getSimpleName(),
            existing.getTypeElement().getQualifiedName().toString());
        return true;
      }
    }

    ...
 }
```
#### 代码生成
在我们收集了所有的有@Factory注解的信息过后，我们可以给每个factory生成代码:
```java:n
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

	...

  try {
        for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
          factoryClass.generateCode(elementUtils, filer);
        }
    } catch (IOException e) {
        error(null, e.getMessage());
    }

    return true;
}
```
写.java文件和写其他文件一样，我们使用`Filer`中提供的Writer来，而写java代码，使用`JavaPoet`或者`JavaWriter`就可以很轻松的办到啦。
```java:n
public class FactoryGroupedClasses {

  /**
   * Will be added to the name of the generated factory class
   */
  private static final String SUFFIX = "Factory";

  private String qualifiedClassName;

  private Map<String, FactoryAnnotatedClass> itemsMap =
      new LinkedHashMap<String, FactoryAnnotatedClass>();

	...

  public void generateCode(Elements elementUtils, Filer filer) throws IOException {

    TypeElement superClassName = elementUtils.getTypeElement(qualifiedClassName);
    String factoryClassName = superClassName.getSimpleName() + SUFFIX;

    JavaFileObject jfo = filer.createSourceFile(qualifiedClassName + SUFFIX);
    Writer writer = jfo.openWriter();
    JavaWriter jw = new JavaWriter(writer);

    // Write package
    PackageElement pkg = elementUtils.getPackageOf(superClassName);
    if (!pkg.isUnnamed()) {
      jw.emitPackage(pkg.getQualifiedName().toString());
      jw.emitEmptyLine();
    } else {
      jw.emitPackage("");
    }

    jw.beginType(factoryClassName, "class", EnumSet.of(Modifier.PUBLIC));
    jw.emitEmptyLine();
    jw.beginMethod(qualifiedClassName, "create", EnumSet.of(Modifier.PUBLIC), "String", "id");

    jw.beginControlFlow("if (id == null)");
    jw.emitStatement("throw new IllegalArgumentException(\"id is null!\")");
    jw.endControlFlow();

    for (FactoryAnnotatedClass item : itemsMap.values()) {
      jw.beginControlFlow("if (\"%s\".equals(id))", item.getId());
      jw.emitStatement("return new %s()", item.getTypeElement().getQualifiedName().toString());
      jw.endControlFlow();
      jw.emitEmptyLine();
    }

    jw.emitStatement("throw new IllegalArgumentException(\"Unknown id = \" + id)");
    jw.endMethod();

    jw.endType();

    jw.close();
  }
}
```
#### 处理器process重复调用
一个处理器的process方法一般会重复调用好几次，因为在第一次运行的时候会生成一些文件，而在编译这些生成的文件前还会去运行处理器，而这重复调用的过程中，同一个处理器始终只有一个对象，而处理器的process方法会反复调用。由于对象没有释放，我们需要去清空上次已经处理过的`FactoryGroupedClasses`
```java:n
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	try {
      for (FactoryGroupedClasses factoryClass : factoryClasses.values()) {
        factoryClass.generateCode(elementUtils, filer);
      }

      // Clear to fix the problem
      factoryClasses.clear();

    } catch (IOException e) {
      error(null, e.getMessage());
    }

	...
	return true;
}
```
关于这个问题的解决办法其实很多，在这里我们使用这个方式，你只要知道注解处理器可能会运行多次就可以了，这样在出现问题的时候知道怎么处理~
#### 处理器代码与工程代码分离
因为处理器的代码是不需要进入工程的，我们需要的最多也是处理器生成的代码，所以可以把处理器代码分开成单独module且不编译进最终的jar或者apk。
### 总结
注解处理器能做的是减少开发时候定式代码的重复书写，解放开发者。

























