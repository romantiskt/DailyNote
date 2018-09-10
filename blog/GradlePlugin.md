---
layout: post
title: "Gradle插件编写与发布和动态修改字节码"
date: 2018-9-6 18:36
comments: true
tags: 
    - 技术
---

---


# GradlePlugin
插件可分为三种方式编写，每种方式各有利弊，根据业务逻辑选择
* Build Script
* BuildSrc Project
* Standalone project
* 动态获取配置
* 修改字节码文件

在大多数情况下，我们都是用groovy来编写插件，因为groovy更简洁优雅，如果你不喜欢或是不太熟悉groovy，你也可以用java实现，插件编写支持groovy和java或者kotlin同时混编

### Build Script
在模块的build.gradle直接编写脚本代码，仅限于当前moudle,不利于复用

* 在模块的build.gradle直接创建一个plugin
```
/**
 * 只可在module的build.gradle定义的plugin
 */
class helloWorldPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('build-hello') {//tasks名称
            group = "android"//tasks所在的组
            description = "gradle build script demo,shares only in this build.gradle"
            doLast {
                println "Hello from the BuildScriptPlugin"
            }
        }
    }
}
```
* 当前module下build.gradle apply
```
apply plugin: helloWorldPlugin
```
### BuildSrc Project
在项目建立一个gradle默认会编译的路径，整个项目的moudled都可通用

* 创建'rootProjectDir/buildSrc/src/main/groovy'
* 编写源码
* apply
```
apply plugin: com.rolan.ProjectBuild//需要写包名+类名
```

### Standalone project
* 创建一个Android Module
```
1.删除除src/main build.gradle文件外所有内容
2.清空build.gradle
3.创建 src/main/groovy/包名 目录
4.创建 src/main/resources/META-INF/gradle-plugins/com.rolan.eventplugin.EventStone.properties //包名+
```
* 添加groovy语言支持
```
apply plugin: 'groovy'

dependencies {
    //gradle sdk
    compile gradleApi()
    //groovy sdk
    compile localGroovy()
}
```
* 编写插件代码
* 发布

1.build.gradle添加如下代码
```
apply plugin: 'maven-publish'

publishing {
    publications {
        mavenJava(MavenPublication) {
            groupId 'com.rolan.eventplugin'
            artifactId 'eventplugin'
            version '1.0.0'
            from components.java
        }
    }
}
publishing {
    repositories {
        maven {
            // change to point to your repo, e.g. http://my.org/repo
            url uri('../maven')//发布到本地目录中
        }
    }
}

```
2.gradlew publish 发布
 ```
   注意：记得发布或者升级版本之前不要配置了项目的 classpath,不然会发布错误
 ```
  
  
3.依赖
```
【project/build.gradle】

 classpath 'com.rolan.eventplugin:eventplugin:1.0.0'
 
 
【app/build.gradle】

apply plugin: com.rolan.eventplugin.EventStonePlugin//需要写包名+类名

```

### 动态获取配置
  我们经常能看到第三方的项目会要求在build.gradle配置一些参数，这些参数有什么作用，怎么获取
  
 * 参数编写
 ```
 class ProjectExtension {//这里首字母一定要大写
     String name = null
     String version = null
 }
 
 class ModuleConfig {
     String name = null
 }
 
 ```
 
 * 参数获取与关联
 ```
  //关联
  project.extensions.create('project_config', ProjectExtension)//这里会关联到build.gradle中配置的参数
  project_config{}
         project.extensions.create('module_config', ModuleConfig)
         
  //获取       
  def project_config=project['project_config']
  def module_config=project['module_config']
  
  println "project_config name:"+project_config.name
  println "project_config version:"+project_config.version
 
 ```
 * 配置
 ```
 apply plugin: com.rolan.ProjectBuild//需要写包名+类名
 ------------>这下面两种方式都可以
 project_config{
     name "GradlePlugin"
     version "0.0.1"
     module_config{
         name "buildSrc module"
     }
 
 }
 
 project_config{
     name "GradlePlugin"
     version "0.0.1"
 }
 module_config{
          name "buildSrc module"
      }
 ```
 
 ### 修改字节码文件
 * Transform的简要书写
 
 这里暂时没有对字节码文件做修改操作，而是将输入转给了输出，这是必须得基本步骤，因为Transform是一个串联的操作，
 必须有输入和输出，不然会中断编译
 ```
 class EventTransform extends Transform {
 
     private Project project
 
     EventTransform(Project project) {
         this.project = project
     }
 
     @Override
     String getName() {//tasks 名称
         return "EventStonePlugin"
     }
 
     @Override
     Set<QualifiedContent.ContentType> getInputTypes() {//处理的文件类型
         return TransformManager.CONTENT_CLASS
     }
 
     @Override
     Set<? super QualifiedContent.Scope> getScopes() {//指定task 作用范围
         return TransformManager.SCOPE_FULL_PROJECT
     }
 
     @Override
     boolean isIncremental() {
         return false
     }
 
     @Override
     void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
         super.transform(transformInvocation)
         transformInvocation.inputs.each {
             it.jarInputs.each {//对类型为jar文件的input进行遍历[第三方依赖]
                 println "*******file_path jarInputs********" + it.file.absolutePath
                 def jarName = it.name
                 def md5Name = DigestUtils.md5Hex(it.file.getAbsolutePath())
                 if (jarName.endsWith(".jar")) {
                     jarName = jarName.substring(0, jarName.length() - 4)
                 }
                 // 获取output目录
                 def dest = transformInvocation.outputProvider.getContentLocation(
                         jarName + md5Name, it.contentTypes, it.scopes, Format.JAR)
                 FileUtils.copyFile(it.file, dest)
             }
             it.directoryInputs.each {
                 //对类型为“文件夹”的input进行遍历[包含书写的类以及R.class、BuildConfig.class以及R$XXX.class等]
                 println "*******file_path directoryInputs********" + it.file.absolutePath
                 // 获取output目录
                 def dest = transformInvocation.outputProvider.getContentLocation(
                         it.name,
                         it.contentTypes,
                         it.scopes,
                         Format.DIRECTORY)
                 FileUtils.copyDirectory(it.file, dest)
             }
         }
     }
 }
 ```
 * 在Plugin中注册
 ```
 void apply(Project project) {
         project.android.registerTransform(new EventTransform(project))
     }
 ```
 * 操作class文件
 
 1.迭代文件夹目录，找出需要修改的class
 ```
  void injectCodeByDir(String path, String rootPath){
         try {
             pool.insertClassPath(path);
         } catch (NotFoundException e) {
             e.printStackTrace();
         }
         File dir = new File(path);
         listFiles(dir,rootPath);
     }
 
     void listFiles(File dir,String rootPath) {
         if (dir.isDirectory()) {
            for(File file:dir.listFiles()){
                listFiles(file, rootPath);
            }
         } else {
             injectCode(dir,rootPath);
         }
     }
 
     void injectCode(File file,String rootPath) {
         String filePath = file.getAbsolutePath();
         if (filePath.endsWith(".class")
                 && !filePath.contains("R$")
                 && !filePath.contains("R.class")
                 && !filePath.contains("BuildConfig.class")) {
             println("****class file****"+filePath);
             int index = FileUtil.isSamePackage(filePath, packageName);
             if(index!=-1){//未设置packageName默认不检查返回true
                 println("***************--------***************"+filePath);
                 String className = filePath.substring(index, filePath.length() - 6)
                         .replace('\\', '.').replace('/', '.');
                 println("***************----className----***************"+className);
 //                addConstructorMethod(rootPath, className);
                 injectMethod(rootPath,className);
             }
         }
 
     }
 ```
 2.为所有类添加构造方法
 ```
  private void addConstructorMethod(String rootPath, String className) {
         CtClass ctClass = null ;
         try {
             ctClass = pool.getCtClass(className);
 
             if (ctClass.isFrozen()) {
                 ctClass.defrost();
             }
             String body = "System.out.println(\"构造方法\" ); ";
             CtConstructor[] cts = ctClass.getDeclaredConstructors();
             if (cts == null || cts.length == 0) {
                 //手动创建一个构造函数
                 CtConstructor constructor = new CtConstructor(new CtClass[0], ctClass);
                 constructor.insertBeforeBody(body);
                 ctClass.addConstructor(constructor);
             } else {
                 cts[0].insertBeforeBody(body);
             }
             ctClass.writeFile(rootPath);
             ctClass.detach();
         } catch (Exception e) {
             e.printStackTrace();
         }
     }
 ```
 3.修改已有方法
 ```
 private void injectMethod(String rootPath,String className) {
         try {
             CtClass ctClass = pool.get(className);
             if (ctClass.isFrozen()) {
                 ctClass.defrost();
             }
             for (CtMethod ctmethod : ctClass.getDeclaredMethods()) {
                 String methodName = FileUtil.getSimpleName(ctmethod);
                 println("----method----"+methodName);
                 if("onClick".equals(methodName)){
                     String body = "System.out.println(\"加入的方法\" ); ";
                     CtMethod method = ctClass.getDeclaredMethod("onClick",new CtClass[]{pool.get("android.view.View")});
                     method.insertAfter(body);
 
                 }
             }
             ctClass.writeFile(rootPath);
             ctClass.detach();
         } catch (Exception e) {
             e.printStackTrace();
         }
     }
 ```
 4.注意
 ```
 project.android.bootClasspath.each {//这里会将android sdk插入进去，因为如果下面需要get android里的类，不然会报notFoundExcetion
             println "android absolutePath: " + it.absolutePath
             pool.appendClassPath(it.absolutePath)
         }
 ```