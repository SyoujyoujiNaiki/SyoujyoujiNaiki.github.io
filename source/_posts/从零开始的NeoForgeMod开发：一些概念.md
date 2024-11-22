---
title: 从零开始的NeoForgeMod开发：一些概念
date: 2024-11-21 16:24:48
tags: [mcmod, dev]
categories: [mcmod]
---

# Mod开发需要使用注册

不管是NeoForge还是Forge还是Fabric等等之类Mod Loader，简单来说都是在MC游戏加载的过程中帮你注入Mod内容。
Minecraft 1.21的游戏物品内容都是通过注册来实现的，也就是说如果我们要添加我们自己的内容进去同样需要进行注册。

![Register](1-1.png)

# 物品（Item）注册

这里我们以一个物品（item）为例，看一下一个基本的物品如何添加进入。

这里为了组织代码，把一些注册的代码分在了不同的包中，一般很多Mod都会这么干。
![Package Hierarchy](1-2.png)

首先我们需要给Item这一类内容创建一个Register，也就是用来注册的工具。这个NeoForge已经帮我们提供了一个DeferredRegister.Item，我们只需要传入我们MOD的ID就可以创建。
如上图的registerSimpleItem是NeoForge提供的简化后的Item注册函数。

```java Item Register
public static final DeferredRegister.Items ITEMS = DeferredRegister.createItems(MODID);
```

这里我们注意到，Register是一个静态成员。实际上物品方块的注册都是仅有一个实例存在的，在我们的注册函数之外的地方都不应该用new再来创建一个实例。

然后注册物品的时候，就需要调用ITEMS的register系列方法来实现：
```java registerSimpleItem
    public static final DeferredItem<Item> EXAMPLE_ITEM = ITEMS.registerSimpleItem(
            "example_item",
            new Item.Properties().food(
                    new FoodProperties.Builder()
                            .alwaysEdible()
                            .nutrition(1)
                            .saturationModifier(2f)
                            .build()
            )
    );
```
这里使用了registerSimpleItem(String, Properties)这个NeoForge提供的简化函数，简化了registerItem(String, Properties -> Item)这个函数，只需要传入Properties即可。
第一个参数String描述了物品的ID，实际上是作为在整个游戏内可以索引的资源路径
Properties是描述Item属性的一个类，我们可以在Properties里面定义Item的一些属性，比如上面代码描述，作为食物的属性，还有其他比如作为武器的伤害属性等等，这里只做注册演示，不一一赘述，待后续章节介绍。
如果不想有这些额外属性，只需要new Item.Propertiers()即可。

完成上述注册后，还有最重要的一步，即，把我们上面创建的Register ITEMS加入到Mod加载中，即在我们的Mod入口源代码文件modid.java的构造方法里加上
```java "Register to mod event bus"
ITEMS.register(modEventBus);
```
这样才彻底完成物品的整个注册流程，否则游戏中将找不到我们注册的物品。

以后如果需要直接使用我们的物品类，一般应当使用DeferredItem<>.get方法获得，如果我们的Item Register没有被注册到mod event bus里面，那么游戏会在遇到请求实例的代码时崩溃并抛出空指针异常。

注册物品之后，其实并没有直接出现在游戏里的创造模式物品标签页（Creative Tab）里，我们需要再手动添加。

把物品添加到创造模式物品标签页有两种方法，一种是在mod event bus里面监听“BuildCreativeModeTabContentsEvent”，如果发现正在创建对应的标签页，则把物品加入其中
```java "Add into creative tab"
    private void addCreative(BuildCreativeModeTabContentsEvent event) {
        if (event.getTabKey() == CreativeModeTabs.BUILDING_BLOCKS)
            event.accept(EXAMPLE_ITEM);
    }
```
上述代码将把EXAMPLE_ITEM添加到“建筑方块”（Building Blocks）这一标签页里。

另一种则是在我们注册创造模式标签页的时候手动添加：

```java "Creative Tab Register"
    public static final DeferredRegister<CreativeModeTab> CREATIVE_MODE_TABS = DeferredRegister.create(Registries.CREATIVE_MODE_TAB, MODID);
```

同样，我们首先声明对应的Register，然后再注册我们的创造模式标签页：

```java "Creative Tab"
    public static final DeferredHolder<CreativeModeTab, CreativeModeTab> NMB_TAB = CREATIVE_MODE_TABS.register("nmb", () -> CreativeModeTab.builder()
            .title(Component.translatable("itemGroup.nullamultiblock")) //The language key for the title of your CreativeModeTab
            .withTabsBefore(CreativeModeTabs.COMBAT)
            .icon(() -> EXAMPLE_ITEM.get().getDefaultInstance())
            .displayItems((parameters, output) -> {
                // Add the example item to the tab. For your own tabs, this method is preferred over the event
                output.accept(EXAMPLE_ITEM.get());
            }).build());
```

在displayItem里面的output添加我们的物品（注意，这里需要使用我们上面注册好的EXAMPLE_ITEM静态实例而不是再new一个），这样可以直接添加到我们自己定义的创造模式标签页里。

进入游戏后，我们可以看到Building Blocks这一标签页最后出现了一个黑紫相交图案的物品，名称是item.modid.example_item
![Example Item](1-3.png)

这就是我们添加的物品，但是我们并没有给他相应的资源（assets），包括本地化（i18n或者叫l10n）和纹理（Texture）。游戏内的资源都统一放置在resources/assets/modid下面。

## 本地化资源

前面我们看到的“item.modid.example_item”这一物品名称是游戏里的ID，“item”和“modid”类似于命名空间，可以让我们定义同名的物品（item）或方块（block）。
要让这个游戏内部使用的名字变成我们在游戏里正常见到的名称，我们需要创建对应的本地化资源文件，位于src/main/resources/assets/modid/lang这一目录下。
本地化资源文件是一个json文件，命名使用语言-国家代码，即2字母的语言代码后用横线再接一个2字母的国家代码，例如美国英语是en_us，中国简体中文则是zh_cn。
这个json文件格式是一个简单的键值对：
```json Key-value
"<id>": "<localization>"
```
这样就可以把对应的游戏内显示的文本进行本地化。如果没有对应语言的本地化文件（比如我们在游戏的语言选项里选择法语），则会发生fallback，比如fallback到en_us（如果你只有zh_cn.json这一个本地化文件，那就只会fallback到这里）。一般推荐都至少有一个en_us.json的本地化翻译。

包括创造模式标签页在内的，一切在游戏里显示的文本都应该通过这种方式来实现本地化，否则只能通过硬编码，导致无法翻译为其他语言。

```json 完整的en_us.json
{
  "itemGroup.nullamultiblock": "Nulla Multiblock Tab",
  "item.nullamultiblock.example_item": "Example Item"
}
```

## 纹理资源

参考上面，要添加纹理，同样要提供对应的纹理文件，这里我们可以直接简单地使用一个png图片来提供。创建一个32x32像素的png文件，命名为<item_id>.png，放置在src/main/resources/assets/modid/textures/item目录下即可自动匹配到对应的item_id上。

但是我们进入游戏会发现此时依然还是紫黑相交的方形，这是纹理缺失的表现。

这是因为，虽然Minecraft表现得非常像素，但是其实依然是一个3D游戏，物品，特别是当你手持物品时，同样是当作一个模型（model）来渲染的（这样就可以实现更复杂的物品，比如可以手持一根精致的法杖或者机枪而不是一个二维像素），所以我们这时还要提供物品的模型描述。

Minecraft对于简单的物品已经提供了最基本的模型描述，只要在src/main/resources/assets/modid/models/item下创建同名的json文件，填入以下内容
```json models
{
  "parent": "item/generated",
  "textures": {
    "layer0": "modid:item/example_item"
  }
}
```
就可以正常显示纹理了。

从文件里的几个键名来看，可以猜到对于这套模型描述，Minecraft也是使用类似继承的机制，通过不同的parent，和剩下的属性可以完成更复杂的模型；textures表示声明我们的纹理；从layer0可以看出纹理还可以叠加。
这里不做更复杂的模型，我们只要最简单的即可在游戏内显示。

![Example Item](1-4.png)