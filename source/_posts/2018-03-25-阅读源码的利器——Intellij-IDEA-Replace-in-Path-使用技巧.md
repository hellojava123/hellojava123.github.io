---
layout: post
title: 阅读源码的利器——Intellij-IDEA-Replace-in-Path-使用技巧
date: 2018-03-25 11:11:11.000000000 +09:00
---


## 前言

讲讲宇宙排名第二的开发工具-----IDEA的使用技巧。

## 搜索/替换 技巧

阅读源码的利器

![](https://upload-images.jianshu.io/upload_images/4236553-3c938312209627f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**1. Match case：** 如果勾选该按钮，搜索时将区分大小写字母。
**2. Preserve case：** 如果勾选该按钮，搜索时不区分大小写，但替换的时候，将会把你给定的字符串的首字母替换成小写。比如，你输入 HelloWorld，将会被替换成 helloWorld。
**3. regex：** 可以使用正则表达式搜索，可参照 java.util.regex。
**4. 右上角蓝色漏斗有几个选项：** 


选项 | 作用
---|---
anywhere  | 选择此选项可随处搜索
in comments  | 选择此选项将搜索限制在注释中，忽略其他事件
In string literals   | 选择此选项可将搜索限制为字符串文字，忽略其他事件
Except *   | 以上三个的反向查找
   

**5. File mask：** 可以过滤要查找的文件格式。可以使用通配符：


通配符  | 作用
---|---
*  | 替换一组任何字符
？  | 替换单个字符
!  | !排除文件。请注意，!应该先以特定文件名称模式进行，例如，!*.gant

可以同时指定多个文件，使用逗号隔开。注意：！，即否定模式，隐式的使用了 * 号匹配。

**6. Search field：** 这是我们使用的最多的，即——搜索框，可手动输入，也可以点击下拉框寻找历史记录。也可以使用正则表达式。

**7. Replace field：**替换字段，可指定替换的文本，也可以使用正在表达式替换文本，如果要在表达式中使用 \，则需要在前面插入三个额外的反斜杠用于转义。

**8. In Project：** 在自己的项目范围中搜索。

**9. Module：**在模块中搜索， 可以指定模块，并可以在下拉框切换模块哦。

**10. Directory：**在指定目录内搜索。右侧那个小文件树 icon，好像并没什么用啊......

**11. Scope：**  在指定范围内搜索。下拉框中有各种范围。


 **12. Preview area：** 当然,最强大的还是预览窗口了，可以使用方向键上下预览，并且可以在预览框中编辑，爽的不行。

**13. 最危险的是下面这个操作：**

![](https://upload-images.jianshu.io/upload_images/4236553-08b01a20d43edede.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当你重构的时候，弄的不好，就全部替换了。。。。。那就尴尬了。
说说上面几个选项的具体作用：

选项 | 作用
---|---
Replace | 替换选中的目标。
Skip | 跳过选中的目标
Replace All in This File | 替换选中的目标所在文件的所有匹配字符串
Skip To Next File | 放弃当前文件，跳到下一个文件
All Files | 这个很危险了。。。替换所有文件
Review | 保险起见，用这个检查每个文件吧。


**关于  Review ：**

![image.png](https://upload-images.jianshu.io/upload_images/4236553-e96e45ff7012ceef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个算是手动模式吧，你可以一个一个检查。Replace All 就是替换所有内容，比较危险，Replace Selected 就是替换选中的内容（使用 ctrl 或 shift 多选）。


## 总结

好了，关于 IDEA 的搜索功能就介绍到这里啦，其实，在阅读源码的过程中，真的要学会善用搜索，当然，不仅是搜索，还有各种功能，比如打断点，断点的跳转，类的继承，实现 UML，方法调用栈，线程调用栈，变量条件判断等等，很多，这些都是阅读源码时不可获取的重要功能，在 debug 的时候，能大大提高我们的效率。

最后，如有条件，请支持正版。谢谢。

goog luck！！！
















































