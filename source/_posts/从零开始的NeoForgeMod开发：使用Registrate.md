---
title: 从零开始的NeoForgeMod开发：使用Registrate
date: 2024-11-29 20:15:56
tags: [mcmod, dev]
categories: [mcmod]
---

# Registrate

前面我们已经做了基本的物品和方块的注册以及对应资源的放置位置，可以看到其实注册一个物品和方块需要手写好几个json文件，还需要多处调用Register。
这里我们介绍使用Registrate这个工具来方便我们的物品注册与资源生成。

Registrate是对Minecraft和NeoForge的注册机制的封装，方便用户注册和自动生成数据，并且只是一个wrapper而不是一个mod，也就是说我们可以直接把他作为一个依赖放在代码里而不需要再要求玩家额外安装一个mod乃至引起mod版本兼容问题。
Registrate最早由[GitHub用户tterrag1098](https://github.com/tterrag1098)开发，但1.21版本后疑似由[IThundxr](https://github.com/IThundxr)接手维护，所以这里我们使用IThundxr提供的maven链接。

# 引入Registrate

在build.gradle的repositories里加入maven地址：
```groovy repositoies
repositories {
    maven { // Registrate
        url "https://maven.ithundxr.dev/snapshots"
    }
    mavenLocal()
}
```
然后再在dependencies里加入Registrate：
```groovy dependencies
dependencies {
    implementation "com.tterrag.registrate:Registrate:${registrate_version}"
}
```
这里的${registrate_version}是一个gradle property，需要在gradle.properties这个文件里添加定义。我们在这里用gradle.properties把这个版本数据与编译脚本分隔开，类似变量定义，防止编译脚本里出现不同版本的引用以及方便数据归类等。
```ini properties
registrate_version=MC1.21-1.3.0+55
```

修改完gradle的编译脚本后，选择使用IDEA的sync gradle project来更新编译依赖，这样就完成了Registrate的引入。

# 声明Registrate

要开始使用Registrate非常简单，我们只需要在我们的Mod入口主文件里面加入一句声明：
```java Registrate
    // Registrate
    public static final Registrate REGISTRATE = Registrate.create(MOD_ID);
```
接下来我们直接使用REGISTRATE就可以开始注册物品和方块等，而不用再一次次地不断创建对应类型的Register了，现在我们可以把之前创建的那些DeferredRegister.Items和BlockDeferredRegister.Blocks代码删除了。
