---
title:        "打造自己的View注解框架"
description:  "向JakeWharton大神致敬，这是一篇模仿ButterKnife的项目的介绍，做这个项目的目的是为了熟悉Annotation Processor，并模仿ButterKnife的功能"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2017-07-05"
---
# 打造自己的View注解框架
## 写在前面的
[ButterKnife](https://github.com/JakeWharton/butterknife)应该都用过或者了解过，没了解快去点电梯。关于ButterKnife的介绍以及使用，GitHub上已经写得很清楚了，我也就不废话了。

那么把ButterKnife在项目中的优劣点简单说一说，首先，作为开发人员来讲，ButterKnife是一款特别方便的框架，它让我们摆脱了大量枯燥的重复代码，把精力专注于业务逻辑，而且使用了ButterKnife后的代码阅读起来更加简洁明了。那么缺点呢，因为ButterKnife用到了反射去给字段赋值，尽管ButterKnife使用了缓存去尽可能地去减少这个过程，但是或多或少都会影响到运行的效率。所以，项目要不要使用ButterKnife就要看各位的取舍了。

但是，不可否认的是，ButterKnife确实是使用特别多，而且也是使用Annotation Processor的经典学习框架。所以，这里我们去模仿ButterKnife打造一个自己的View注解框架。当然我们的这个框架只是学习为主，并不推荐用到项目中去。

而关于Annotation Processor的内容，送你电梯[注解处理器](http://www.shie1d.com/2017/06/28/Annotation-Processing/)

而这篇文章对应的项目戳这里[ViewTrick-Project](https://github.com/ShieldShen/ViewTrick-Project)
## 关于框架的结构
含有Annotation Processor的部分，我们不需要把他编译到代码里面去，所以作为一个单独的模块拿出来。而功能的使用，ViewTrick模块，作为一个面向三方的模块独立出来。而前面说的两个模块都要用到相同的注解，所以注解也会放在一个单独的模块当中。
![](/resources/img-post/viewtrick/viewtrick_1.png)
那么用户在使用我们的框架的时候只需要在dependencies中添加

```
    apt project(':viewtrick-compiler')
    compile project(':viewtrick')
```

而apt的使用也在[注解处理器](http://www.shie1d.com/2017/06/28/Annotation-Processing/)有介绍
## 功能实现的大概思路
我们需要给View字段注入，一般的流程是使用`View#findViewById(int)`来从视图树中获取。所以，我们需要使用者给我们提供我们要注入的View字段的**Id**以及这个View所属的**视图树**。

Android中所有的`Window`都会先添加一个`DecorView`然后向`DecorView`中添加布局，所以，需要从用户的传参里面拿到我们要注入的View所属的根。这个就是我们需要的*视图树*。而*id*我们可以通过使用者给我们。

1. 获取id：一个View对应一个id，而我们对一个View的注入就是依赖这个id来获取，这个可以通过注解来让使用者告诉我们。
2. 获取视图树：在`Activity`中，我们可以获取`Window`并通过它来获取`DecorView`

### 从bind方法说起
从使用者的代码里，就像这样

```java
...
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.tv)
    public TextView tv;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ViewTrick.bind(this);
    }
...
}
```

就可以对`tv`进行赋值。

而`ViewTrick#bind(T target)`这个方法是使用者在执行代码里面添加的唯一一句代码，我们需要在`bind`方法里面完成给所有View注入的任务。

当然在这里，我们完全可以通过全部使用反射来达到给View注入的目的，但是这样对性能的损耗是相当大的，而且我们知道了注解处理器这个东西，就可以把运行时执行的反射操作转换成编译时的生成代码去给View注入。

所以在`bind`中，我们只需要去调用我们生成的类去给字段赋值。

```java
    public static void bind(Activity activity) {
        View decorView = activity.getWindow().getDecorView();
        try {
            Class<?> clz = Class.forName(activity.getClass().getCanonicalName() + "$$ViewBinder");
            Constructor<?> constructor = clz.getConstructor(View.class, activity.getClass());
            constructor.newInstance(decorView, activity);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
```

虽然这里也用到了反射，但这里的反射是去寻找我们生成的类，对于一个视图而言，我们只需要反射调用一次，然后调用类的构造方法来给View赋值。相比于对每个字段使用反射去赋值来说，这样的效率要提升太多太多。
### 关于注解
使用者通过注解告诉我们哪些字段需要注入，并且向注解中写入字段的id，我们通过这个id去寻找View

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface BindView {
    int value();
}
```

### 关于注解处理器
那么，现在我们主要关注的地方就在注解处理器生成的代码上。我们对于每个需要注入的视图，回去生成一个对应的类，这个类的构造方法里面会对使用者注解了的视图字段进行赋值。所以生成的代码就像这样

```java
public class MainActivity$$ViewBinder<T extends MainActivity> {
  public MainActivity$$ViewBinder(View view, T target) {
    target.tv = (TextView)view.findViewById(2131427415);
  }
}
```

那么，我们明确了这个生成的类是要做什么的之后，我们就可以根据注解去生成代码了。

当然，为了避免用户在我们不能处理的字段上添加注解，所以我们要提前设立一些规则：
1. 我们要在外部使用到这些字段，所以这些字段不能为`private`
2. 字段必须是View的子类
3. 字段所存在的类不能是接口以及抽象类(其实这里可以处理，只是为了处理简单设立了这么一个规则)

#### 处理器中需要的一些数据对象：
##### 被注解的字段以及相关信息*BindViewField*
```java
public class BindViewField {
    public VariableElement element;
    public int resId;

    @Override
    public String toString() {
        return "BindViewField{" +
                "element=" + element +
                ", resId=" + resId +
                '}';
    }

    public BindViewField(VariableElement element) {
        this.element = element;
        String fieldName = element.getSimpleName().toString();
        BindView annotation = element.getAnnotation(BindView.class);
        if (annotation == null || annotation.value() == 0) {
            throw new IllegalArgumentException(fieldName + " don't has a @BindView annotation nor value of annotation is missing");
        }
        resId = annotation.value();
    }
}
```

其中`VariableElement`表示被注解的字段元素，而resId就是注解中的资源id值。
##### 含有被注解字段的类*BindViewContainer*

```java
public class BindViewContainer {
    private LinkedHashMap<String, BindViewField> boundFields;
    private String containerName;

    public BindViewContainer(String containerName) {
        this.containerName = containerName;
        boundFields = new LinkedHashMap<>();
    }

    public void put(BindViewField field) throws IllegalArgumentException {
        String fieldName = field.element.getSimpleName().toString();
        BindViewField bindViewField = boundFields.get(fieldName);
        if (bindViewField != null) {
            throw new IllegalArgumentException("Duplicate name " + fieldName + " used in " + containerName);
        }
        boundFields.put(fieldName, field);
    }

    public void generateCode(Filer filer, Elements elementUtils, Types typeUtils) throws IOException, ClassNotFoundException {
        TypeElement containerElement = elementUtils.getTypeElement(containerName);
        PackageElement packageElement = elementUtils.getPackageOf(containerElement);
        String packageName = packageElement.getQualifiedName().toString();
        String[] split = containerName.split("\\.");
        String clzName = split[split.length - 1];


        String paramView = "view";
        String paramTarget = "target";
        String typeT = "T";
        TypeVariableName genericType = TypeVariableName.get(typeT, TypeName.get(containerElement.asType()));
        ParameterSpec constructorParamView =
                ParameterSpec.builder(
                        TypeName.get(elementUtils.getTypeElement("android.view.View").asType()), paramView
                ).build();
        ParameterSpec constructorParamTarget =
                ParameterSpec.builder(
                        genericType, paramTarget
                ).build();


        MethodSpec.Builder builder = MethodSpec.constructorBuilder()
                .addModifiers(Modifier.PUBLIC)
                .addParameter(constructorParamView)
                .addParameter(constructorParamTarget);
        for (BindViewField field :
                boundFields.values()) {
            builder.addStatement("$N.$N = ($T)$N.findViewById(" + field.resId + ")", paramTarget, field.element.getSimpleName(), ClassName.get(field.element.asType()), paramView);
        }
        MethodSpec constructor = builder.build();
        TypeSpec typeSpec = TypeSpec.classBuilder(clzName + "$$ViewBinder")
                .addModifiers(Modifier.PUBLIC)
                .addTypeVariable(genericType)
                .addMethod(constructor)
                .build();
        JavaFile.Builder fileBuilder = JavaFile.builder(packageName, typeSpec);
        fileBuilder.build().writeTo(filer);
    }

    @Override
    public String toString() {
        return "BindViewContainer{" +
                "boundFields=" + boundFields +
                ", containerName='" + containerName + '\'' +
                '}';
    }
```

其中`containerName`表示包含被注解字段的类的全路径名，`boundFields`记录这个类里面包含的所有被注解的字段。

#### 校验代码
这里是根据之前的规定写的校验字段是否符合要求的，代码在仓库里有，所以这里不做讲述，api的学习请到[注解处理器](http://www.shie1d.com/2017/06/28/Annotation-Processing/)。

#### 其中一些关键代码的介绍
##### 通过注解的VariableElement获取字段的类型元素

```java
 TypeMirror typeMirror = boundElement.asType();
 TypeElement typeElement = (TypeElement) mTypeUtils.asElement(typeMirror);
```

我们生成的代码，需要获取注解字段的类名，而类名是通过VariableElement获取不到的，所以要获取相应的TypeElement

##### 通过VariableElement获取字段所在的类

```java
TypeElement containerElement = (TypeElement) boundElement.getEnclosingElement();
```

因为我们生成的类是要与被注解的字段所在的类一一对应的，获取containerElement就很有必要。

##### 生成泛型
 
生成的代码里面有两个地方有泛型:

```java
public class MainActivity$$ViewBinder<T extends MainActivity> {
  public MainActivity$$ViewBinder(View view, T target) {
    target.tv = (TextView)view.findViewById(2131427415);
  }
}
```

所以对应的，在构造方法中参数的泛型:

```java
...
String typeT = "T";
TypeVariableName genericType = TypeVariableName.get(typeT, TypeName.get(containerElement.asType()));
ParameterSpec constructorParamTarget =
                ParameterSpec.builder(
                        genericType, paramTarget
                ).build();   
MethodSpec.Builder builder = MethodSpec.constructorBuilder()
                .addParameter(constructorParamTarget);
...
```

在类的声明中的泛型:

```java
String typeT = "T";
TypeVariableName genericType = TypeVariableName.get(typeT, TypeName.get(containerElement.asType()));
TypeSpec typeSpec = TypeSpec.classBuilder(clzName + "$$ViewBinder")
                .addModifiers(Modifier.PUBLIC)
                .addTypeVariable(genericType)
                .addMethod(constructor)
                .build();
```

> 说明:这个里面的`containerElement`就是注解字段所在的类的元素

### 最后
其实博主很懒，所以没有写代码的详细说明。这个文章主要说明的是这种注入框架的思路，而注解处理的原理也可以电梯去看。而且这个demo只能作为学习哦，用到项目里面出现的各种问题，我表示#￥@%@#

对应的项目戳这里[ViewTrick-Project](https://github.com/ShieldShen/ViewTrick-Project)


















