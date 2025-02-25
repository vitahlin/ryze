
# JDK

通过 sdkman 安装即可，sdkman可以参考：[https://sdkman.io/install](https://sdkman.io/install)
jdk8可以安装 zulu 版本。
maven 也直接通过 sdkman 安装。 

# Intellij-IDEA

## 通用配置

自动格式化：Action on save
显示全部文件：tab：Editor-General-Editor Tabs

## 插件

-  GitToolBox git增强
-  Rainbow Brackets 括号亮色
-  RestfulTool 搜索接口地址
-  Maven Helper 包冲突解决
-  GsonFormatPlus json转bean
-  SonarLint 代码格式化检测
-  Jump to Line 允许您转到任意行并设置执行点而无需执行前面的代码。
-  Grep Console，展示不同颜色日志
-  JRebel 热部署
-  SequenceDiagram 方法调的深度，生产时序图
- String Manipulation 字符串操作，快捷转换大小写

### JRebel

热部署插件，需要激活，需要先激活，激活后可以通过热部署调试。
重新编译快捷键：`Command+Shift+F9`

### Grep Console

终端不同级别的日志配置展示不同颜色。

![image.png](https://vitahlin.oss-cn-shanghai.aliyuncs.com/images/blog/2022/07/202212191442819.png)

从上到下的颜色配置依次是：
- 9B070C
- FF2D2F
- FFCA69
- 987478
- 808080
- 499790