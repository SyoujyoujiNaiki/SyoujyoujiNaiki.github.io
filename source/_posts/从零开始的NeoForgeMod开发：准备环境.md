---
title: 从零开始的NeoForgeMod开发：准备环境
date: 2024-11-21 14:01:55
tags: [mcmod, dev]
categories: [mcmod]
---

# 从零开始的NeoForgeMod开发：准备环境

## 前言

本教程将基于Nulla Multiblock这个mod的开发过程，展示NeoForge-1.21.1 MCMOD的开发细节。其中将尽量不使用其他mod api来开发，方便读者更好理解。

## 开发环境

本教程所开发对应的MC版本为1.21.1，Mojang官方分发的是Java 21，所以需要安装的JDK版本为21。
NeoForge推荐使用的OpenJDK发行版是巨硬的发行版，但这里使用的是Liberica OpenJDK，读者也可以自行选择其他OpenJDK发行版，比如Eclipse Temurin或者对应OS打包的OpenJDK，这里不再赘述，也不提供任何分发。

MC Mod开发同样是Java开发，这里推荐使用的Java IDE是JetBrains IDEA，NeoForge官方的开发模板为IDEA和Eclipse都作了适配，但是官方说其他IDE也是可以用的，只要有Java和Gradle的适配。

## 创建项目

开发基于NeoForge的mod需要使用NeoForge的项目模板：NeoForge提供了[ModGradleDev](https://github.com/NeoForgeMDKs/MDK-1.21-ModDevGradle)和[NeoGradle](https://github.com/NeoForgeMDKs/MDK-1.21-NeoGradle)两种，ModGradleDev是更新的模板，建议使用这个。

在GitHub的项目页面上我们可以看到右上角有一个“Use this template”按钮:
![New Repo](0-1.png)
使用这个可以直接帮我们创建一个新的仓库：
![New Repo](0-2.png)
然后我们再通过Git克隆到本地进行开发。

如果不想直接在GitHub上创建仓库的话，也可以直接把源码仓库下载压缩包到本地，解压到自己的项目文件夹下。

然后用IDEA打开项目文件夹，此时IDEA应该就可以自动根据项目下的build.gradle和gradle.properties文件来配置项目环境。等待配置完毕，就可以直接运行Gradle Tasks里面的runClient来启动MC。
![runClient](0-3.png)

游戏加载完毕后，为了方便调试，可以直接选择创造模式超平坦来进入世界。此时打开背包界面，应该可以看到有一个新的紫黑镶嵌纹理的标签页，这就是我们的mod添加的内容，其中包括一个同样没有纹理的物品。
![mod content](0-4.png)

这些内容是由ExampleMod.java里面实现的，一般来说习惯会把Mod的入口放在与Mod同名的java源文件里面，这里可以自行重命名为我们自己的Mod名字，里面的一些MODID等静态常量也可以修改为我们自己的Mod内容。

Mod开发环境默认使用gradle工具，这里我们大概了解一下：
 - build.gradle，这是用groovy脚本写的gradle编译脚本，控制编译过程；
 - gradle.properties，这是一个键值对的属性文件，里面可以存储一些由build.gradle脚本引用的变量，这样可以方便我们修改一次变量而同步build.gralde里面的所有引用。

我们这里也要修改一下gradle.properties里面的值，把这个Example Mod改成我们自己的Mod
![Properties Content](0-5.png)

现在我们的Mod开发环境就准备完毕了。