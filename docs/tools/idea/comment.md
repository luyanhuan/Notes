# 如何使用自定义的注释模板
今天刚好看到公众号推送的一篇文章，主要讲解了如何使用idea配置自己的注释模板，特来记录一下。
## 快捷键小tip
之前所有的设置都是点击File->Settings，然后在界面里搜，这篇文章里我看到一个更方便哒！直接ctrl+shift+a，Actions里直接搜关键字，
相关的东西就全出来啦！。
## File and Code Templates
Settings里找到该项，在File and Code Templates->Files下找到class文件对应的模板：
```
#if (${PACKAGE_NAME} && ${PACKAGE_NAME} != "")package ${PACKAGE_NAME};#end
#parse("File Header.java")
public class ${NAME} {
}
```
这里我们主要关注#parse("File Header.java"),这块就是生成类注释的，那么这个.java文件在哪呢？File and Code Templates->includes可以找到！
直接按照他要求的格式贴入自定义注释，那么当你新建class文件时会自动加上自定义类注释。
```
/**
 * <p>Title: ${NAME}</p>
 * <p>Description:</p>
 * @auther ${USER}
 * @version V1.0
 * @date ${DATE} ${TIME}
 */
```
## live templates
### 类
除了新建，其实我们也会希望能动态地在类上或者方法上增加自定义注释，这个就要设置live template了！
直接通过快捷键找到live templates，新增一个live template。
![insert live template](/images/tools/idea/1.png)   
然后通过edit variables设置变量值。
![edit variables](/images/tools/idea/2.png)   
### 方法
新增方法的live template与类的大同小异，主要关注param,需要用到Groovy
![insert live template](/images/tools/idea/3.png)  

![edit variables](/images/tools/idea/4.png)  
```
groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {result+=' * @param ' + params[i] + ((i < params.size() - 1) ? '\\n' : '')}; return result", methodParameters())
``` 
大功告成，之后就可以通过快捷键cc和mc直接在类上或者方法上动态添加自定义注释啦！不过方法这块有个坑，就是必须要在方法内部使用mc快捷键添加自定义注释，否则params和return都可能是null