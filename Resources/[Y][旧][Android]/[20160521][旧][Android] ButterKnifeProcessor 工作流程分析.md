## 备注
**原发表于2016.05.21，资料已过时，仅作备份，谨慎参考**

## 前言
在 [[Android] ButterKnife 浅析](http://blog.qiji.tech/archives/10522) 中，我们了解了 ButterKnife 的用法，比较简单。

本次文章我们来学习一下 ButterKnife 的 ButterKnifeProcessor 注解处理器，注解处理器能够解析代码中的注解信息，生成相应的 Java 类，这也是 ButterKnife 的关键实现原理。

建议在阅读前先了解下 Java 中『注解』的概念。

## 准备内容
### APT
APT（Annotation processing tool）是在编译时，扫描和处理注解的一个构建工具，可以在编译源代码时额外生成 Java 源代码。

### AbstractProcessor
AbstractProcessor 是扫描和处理注解的关键类，ButterKnife 自定义的 Processor 就需要继承自该类。

代码示例如下：

    public class MyProcessor extends AbstractProcessor {
        // 在 Processor 创建时调用并执行的初始化操作
        @Override
        public synchronized void init(ProcessingEnvironment env){ }
    
        // 关键方法，进行扫描和处理注解，并生成新的源代码
        @Override
        public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }
    
        // 指定需要注册的注解
        @Override
        public Set<String> getSupportedAnnotationTypes() { }
        
        // 指定支持的 Java 版本
        @Override
        public SourceVersion getSupportedSourceVersion() { }
    
    }

### 添加依赖
还记得我们使用 ButterKnife 前所做的添加吗？

    dependencies {
      classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
    
    dependencies {
      compile 'com.jakewharton:butterknife:8.0.1'
      apt 'com.jakewharton:butterknife-compiler:8.0.1'
    }

这里 com.jakewharton:butterknife-compiler 就是自定义的注解处理器，我们在 Gradle 中注册使用它。

然而我在项目结构中找了很久也没有找到这个库的文件，有可能是在编译时才去访问的，如果需要可以在 GitHub 中找到：

[butterknife-compiler](https://github.com/JakeWharton/butterknife/tree/master/butterknife-compiler)

## ButterKnifeProcessor 工作流程
上面注册完自定义的注解处理器后，我们就知道，在编译源代码时，APT 会调用 Processor 来查找解析注解。

### 主要逻辑
这里主要看 process 处理方法的内容。

    @Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
        
        // 查找并解析注解
        Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);
        
        // 去除注解的键值
        for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
          TypeElement typeElement = entry.getKey();
          BindingClass bindingClass = entry.getValue();

        // 生成源代码
          try {
            bindingClass.brewJava().writeTo(filer);
          } catch (IOException e) {
            error(typeElement, "Unable to write view binder for type %s: %s", typeElement,
                e.getMessage());
          }
        }
      return true;
    }

可以看到 process 方法的逻辑还是比较容易理解的，就是获取到注解的键值并生成源代码。

### 查找和解析注解
首先看 findAndParseTargets(RoundEnvironment env) 方法，通过参数 RoundEnvironment env 可以找到我们想要的某一个被注解的元素。

该方法会查找所有 ButterKnife 的注解来进行解析，我们选择最简单的 @BindInt 来看一下：

    private Map<TypeElement, BindingClass> findAndParseTargets(RoundEnvironment env) {
        Map<TypeElement, BindingClass> targetClassMap = new LinkedHashMap<>();
        Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

        // Process each @BindInt element.
        for (Element element : env.getElementsAnnotatedWith(BindInt.class)) {
          if (!SuperficialValidation.validateElement(element)) continue;
          try {
            parseResourceInt(element, targetClassMap, erasedTargetNames);
          } catch (Exception e) {
            logParsingError(element, BindInt.class, e);
          }
        }
        ...

这里调用了 parseResourceInt 方法传入了被注解的元素进行解析：

      private void parseResourceInt(Element element, Map<TypeElement, BindingClass> targetClassMap,
          Set<TypeElement> erasedTargetNames) {
        boolean hasError = false;
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
    
        // Verify that the target type is int.
        if (element.asType().getKind() != TypeKind.INT) {
          error(element, "@%s field type must be 'int'. (%s.%s)", BindInt.class.getSimpleName(),
              enclosingElement.getQualifiedName(), element.getSimpleName());
          hasError = true;
        }
    
        // Verify common generated code restrictions.
        hasError |= isInaccessibleViaGeneratedCode(BindInt.class, "fields", element);
        hasError |= isBindingInWrongPackage(BindInt.class, element);
    
        if (hasError) {
          return;
        }
    
        // Assemble information on the field.
        String name = element.getSimpleName().toString();
        int id = element.getAnnotation(BindInt.class).value();
    
        BindingClass bindingClass = getOrCreateTargetClass(targetClassMap, enclosingElement);
        FieldResourceBinding binding = new FieldResourceBinding(id, name, "getInteger", false);
        bindingClass.addResource(binding);
    
        erasedTargetNames.add(enclosingElement);
      }

方法分为前后两部分，先进行了一些检验，检验通过则调用 getOrCreateTargetClass 获取或生成一个类放入数组中。

isInaccessibleViaGeneratedCode 方法检验了：

1. 方法修饰符不能为 private 和 static
2. 包类型不能为非 Class
3. 类的修饰符不能为 private 

isBindingInWrongPackage 则检验了包名，不能以 android 或 java 开头。

### 生成类文件
解析完每个被注解的元素之后会得到一个 Map<TypeElement, BindingClass> targetClassMap，接着就会调用 Map 中每个 bindingClass 生成 Java 源代码：

    bindingClass.brewJava().writeTo(filer);

brewJava() 的内容如下所示，添加了方法，父类，接口等信息，最后生成了一个 JavaFile，简单了解即可：

    JavaFile brewJava() {
        TypeSpec.Builder result = TypeSpec.classBuilder(generatedClassName)
            .addModifiers(PUBLIC);
        if (isFinal) {
          result.addModifiers(Modifier.FINAL);
        } else {
          result.addTypeVariable(TypeVariableName.get("T", targetTypeName));
        }
    
        TypeName targetType = isFinal ? targetTypeName : TypeVariableName.get("T");
        if (hasParentBinding()) {
          result.superclass(ParameterizedTypeName.get(parentBinding.generatedClassName, targetType));
        } else {
          result.addSuperinterface(ParameterizedTypeName.get(VIEW_BINDER, targetType));
        }
    
        result.addMethod(createBindMethod(targetType));
    
        if (isGeneratingUnbinder()) {
          result.addType(createUnbinderClass(targetType));
        } else if (!isFinal) {
          result.addMethod(createBindToTargetMethod());
        }
    
        return JavaFile.builder(generatedClassName.packageName(), result.build())
            .addFileComment("Generated code from Butter Knife. Do not modify!")
            .build();
    }

于是我们编译过后就能在项目中找到类似 MainActivity$$ViewBinder 这样的文件。

到这里 ButterKnifeProcessor 的工作就结束了，ButterKnife 就能够调用生成的类来进行绑定工作，我们将在下一篇文章中对余下的流程进行学习。

## 参考资料
[AndroidSutdio 编译时自动生成源代码](http://www.septenary.cn/2015/12/19/AndroidSutdio-%E7%BC%96%E8%AF%91%E6%97%B6%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E6%BA%90%E4%BB%A3%E7%A0%81/)

[Android 浅析 ButterKnife (二) 源码解析](http://www.jianshu.com/p/a5ea2ea75bb7)