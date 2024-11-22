---
title: 从零开始的NeoForgeMod开发：再进一步
date: 2024-11-22 16:52:16
tags: [mcmod, dev]
categories: [mcmod]
---

# 添加方块

要添加能够在世界里放置的方块，跟添加物品的流程是类似的，首先需要一个Register，NeoForge也已经提供了：
```java "Block Register"
    public static final DeferredRegister.Blocks BLOCKS = DeferredRegister.createBlocks(MODID);
```

然后对于简单的方块，我们也可以用类似registerSimpleItem的方法来添加：

```java "Example Block"
    public static final DeferredBlock<Block> EXAMPLE_BLOCK = 
            BLOCKS.registerSimpleBlock("example_block", BlockBehaviour.Properties.of());
```

这里也是简化的注册函数，BlockBehaviour.Properties与Item.Properties类似，也是定义一些基本的方块属性的，比如破坏抗性，破坏时间，等等。其中的of()方法还可以用来复制已有的其他方块的属性，这里不多讲解。

这里注册的同样是全局唯一的方块实例，后续在代码中需要引用方块都应该从这里获取，而不是再new一个实例使用。在注册阶段之外创建任何方块实例都可能引发不确定的行为和崩溃。

最后不要忘记了在我们Mod的加载事件中加入这个Register：

```java "Register to the modEventBus"
    // Register the Deferred Register to the mod event bus so blocks get registered
    BLOCKS.register(modEventBus);
```

# 添加方块物品（Block Item）

需要注意一点的是，我们上面添加的仅仅是世界里的方块，而不是可以拿在手上、放在背包里的方块物品。
方块物品同样也是物品，所以我们只需要按添加物品的方式来添加即可，NeoForge也有专门的registerBlockItem方法来添加一般的方块物品。
```java "Block Item"
    public static final DeferredItem<BlockItem> EXAMPLE_BLOCK_ITEM = ITEMS.registerSimpleBlockItem("example_block", EXAMPLE_BLOCK);
```
第一个参数是这个物品的ID，也即资源路径，而第二个参数就是我们上面注册的方块。

方块物品由于是专门对应一个方块的特殊物品，所以一般我们只要给方块资源就可以自动获得对应方块物品的资源。

# 添加资源

## 本地化

与物品一样，对应方块的本地化键名是“block.modid.example_block”，我们只要翻译这一条目，我们的方块物品也会自动显示这条翻译。

## 纹理

与物品一样，方块也需要在assets下有对应的资源文件，纹理图片在src/main/resources/assets/nullamultiblock/textures/block下，模型在src/main/resources/assets/nullamultiblock/models/block下。
这里我们的纹理图片直接复制Example Item的纹理图片。
而模型也是类似物品的json文件：
```json "Block Model"
{
  "parent": "block/cube_all",
  "textures": {
    "all": "modid:block/example_block"
  }
}
```

我们可以注意到textures下有一个键名为"all"，这表示我们这个方块的所有6个面都使用同样的纹理

需要注意的是，方块物品同样也需要提供纹理，但是作为特殊的专门对应方块的物品，可以直接复用方块的资源，这里我们可以不必提供方块物品的纹理图片，只需要给我们的方块物品提供模型文件。
上一章讲过，物品的模型资源目录在src/main/resources/assets/nullamultiblock/models/item下：

```json "Block Item Model"
{
  "parent": "modid:block/example_block"
}
```

这里我们可以看到parent正是我们的方块模型文件，这样就可以直接复用我们的方块模型，就像一般的泥土、圆石等方块一样。
（其实你也可以使物品模型与方块不一样，比如炼药锅就是一例。）

## Block State

完成上述工作后，在游戏内，我们的Example Block其实并未正确加载纹理，因为方块与物品不同，还有一个名为Block State的属性。

Block State顾名思义就是方块的状态：有时候我们可能会需要同一个方块有不同的可数个的状态（比如桶的不同朝向有不同的纹理和桶打开时候纹理的变化），这时候就需要Block State的定义来方便渲染方块。

Block State的描述文件放置在src/main/resources/assets/nullamultiblock/blockstates下，我们创建与Example Block同样id的json文件：

```json blockstate
{
  "variants": {
    "": {"model": "nullamultiblock:block/example_block"}
  }
}
```

我们可以看到里面有variants这一键名，但是我们创建的example_block目前并没有多个状态，我们只需要一个纹理就够了。

完成上述内容后，再进入游戏，我们就可以看到Example Block了：

![Example Block](2-1.png)