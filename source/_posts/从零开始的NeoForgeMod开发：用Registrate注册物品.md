---
title: 从零开始的NeoForgeMod开发：用Registrate注册物品
date: 2024-11-29 21:11:43
tags: [mcmod, dev]
categories: [mcmod]
---

# 创造模式标签页

先给Mod创建一个标签页，只需一个声明：
```java CreativeModeTab
    static {
        REGISTRATE.defaultCreativeTab("nullamultiblock",
                builder -> builder
                        .title(Component.translatable("itemGroup.nullamultiblock"))
                        .icon(Items.WHITE_WOOL::getDefaultInstance)
        ).register();
    }
```
这里首先注意，Registrate注册过程都是静态的，所以我们这里必须在调用register()的时候得是静态方法，也就是在外面包上一层static。
使用static时需要注意，static修饰的声明和方法调用具有先后顺序，代码会从上到下的顺序调用，一些注册过程中如果先后顺序调换，就会出现注册不如预期乃至报错的可能。

调用defaultCreativeTab(String, Comsumer<CreativeModeTab.Builder>)，第一个参数是Tab的Id，第二个参数是一个函数：参数为创造模式标签页Builder、不返回结果的函数，相当于提供了builder给我们进行自定义。
这里我们定义了创造模式标签页的标题（title）和图标（icon）。
与之前不同的是，这里的builder不需要调用build()，Registrate会自动帮我调用。
最后不要忘记了register()来完成注册。

使用Registrate更方便的一点是，我们在这之后再用Registrate来注册物品，那么物品就会自动默认添加在我们刚刚声明的创造模式标签页里，不需要再在相关创造模式标签页的声明或者Mod注册事件里手动添加了。
也由此这段声明一般放在注册物品的代码同文件前面，以保证Registrate先默认使用了我们的创造模式标签页（由于我们之前所说的static修饰的特性）。

# 注册基本物品

注册一个没有其他功能，仅做配方材料之类的，这样一个基本物品，使用Registrate同样也是一个声明就可以完成：
```java
    public static final ItemEntry<Item> RAW_CUPRITE = REGISTRATE.item("raw_cuprite", Item::new).register();
```

ItemEntry<T extends Item>是类似DefferedRegistry的注册条目，用于我们注册后再引用该物品。
REGISTRATE.item(String, Item.Properties -> T)则是我们创建物品的方法，第一个参数是物品的ID，第二个参数是一个函数：参数为Item.Properties，然后返回我们的物品T结果，这样我们就可以直接使用Item的new方法来填入。
最后调用register()我们就完成了物品注册，并且自动添加到了我们在上一小节创建的创造模式标签页里，相比原来的方法，代码可谓简洁了不少。

接下来我们需要再提供物品贴图，这个也只能由我们自己来画，放置在src/main/resources/assets/<modid>/textures/item目录下，与我们的物品Id同名。
但是model文件和lang条目我们则可以使用Registrate来自动生成，无需手动填入！
这里我们要做的是运行gradle任务：runData
（需要注意的是自动生成model json和lang json需要我们首先添加物品的贴图纹理文件，否则会提示找不到对应的资源）。
运行完成后在我们的src/generated/resources/assets/<modid>/目录下就有对应的json文件：
model json文件默认为item/generate这一基本的物品模型；
lang json文件里则默认只提供基本的英文本地化和一个趣味性的倒英文字母本地化，其中会根据我们的物品ID，把_替换为空格，物品单词首字母大写，自动帮我们完成本地化！

运行完runData后我们就可以进入游戏看到我们注册的物品了。

# 分离在Items注册文件下

如果我们想要把Registrate代码像最开始那样分离在一个单独的Items注册文件下，那么由于类没有被引用，此时类里的静态代码不会被加载，也就造成我们的注册无效。
那么此时我们只需要在这个注册代码的最下方加入一个空的静态方法，比如我们创建一个register方法（或者你喜欢也可以叫init之类的）：
```java "static register"
    public static void register() {}
```
然后在我们的Mod入口主文件的构造函数里，直接调用这个register静态方法即可导入。

# 更复杂的物品

接下来我们注册一个更复杂的物品：
```java
    public static final ItemEntry<TieredItem> EXAMPLE_WEAPON =
            REGISTRATE.item("example_weapon", properties -> new TieredItem(
                            Tiers.GOLD,
                            properties
                                    .component(
                                            DataComponents.TOOL,
                                            new Tool(List.of(Tool.Rule.minesAndDrops(List.of(Blocks.GRASS_BLOCK), 2.0f)), 1.0F, 1)
                                    )
                                    .attributes(ItemAttributeModifiers.builder().add(
                                                    Attributes.ATTACK_DAMAGE,
                                                    new AttributeModifier(
                                                            BASE_ATTACK_DAMAGE_ID, 2.0, AttributeModifier.Operation.ADD_VALUE
                                                    ),
                                                    EquipmentSlotGroup.MAINHAND
                                            )
                                            .add(
                                                    Attributes.ATTACK_SPEED,
                                                    new AttributeModifier(BASE_ATTACK_SPEED_ID, 2.0, AttributeModifier.Operation.ADD_VALUE),
                                                    EquipmentSlotGroup.MAINHAND
                                            ).build()
                                    )
                    )
            ).lang("A New Hope").model((ctx, pvd) -> pvd.handheld(ctx)).register();
```
这里我们注册的一个物品是带有工具等级的TieredItem，构造函数是TieredItem(Tier, Properties)，所以我们这里不能直接用他的new方法，而是用一个λ来构建。
Tiers是一个实现Tier接口的枚举，我们这里使用Tiers.GOLD这个值来提供工具等级，工具等级包含了工具的基础耐久，基础挖掘速度，基础攻击伤害等属性；
接下来的物品属性则是重要的自定义部分：component方法是提供DataComponent使用的，attributes则提供了更多的属性修饰。

DataComponent是用于高版本MC开始替代NBT的一个新体系，可以用于物品存储数据使用，比如这里我们使用的DataComponents.TOOL就是DataComponents这个已经事先实现好的DataComponent枚举中用来提供“工具”数据的。
Tool就是其中的数据record，里面包含了一个List<Tool.Rule>和挖掘速度和伤害，而其中的Tool.Rule又是一个record，其中记录的是挖掘特定的一系列方块，是否提供特殊挖掘速度加成并是否正确掉落物品，除了minesAndDrops这个允许挖掘并自定义挖掘速度的规则之外还有deniesDrops这一无法挖掘和禁止掉落的挖掘规则等其他规则，这里不做列举。
DataComponent可以实现的功能远不止于此，因为他只是可以存储数据，那么我们可以通过在里面存储不同的数据，来实现存储物品或者实现一套自己的魔法体系均有可能。
record是一个不可变的数据记录，DataComponent使用时就需要其中包含的数据不可变，这点在使用时需要注意，不能在读取DataComponent内的数据时对其进行修改。
由于DataComponent内容较长，这里不再做更多介绍，如有机会，后续文章会继续讲解。

Attribute则是物品提供的属性（详细可参阅[Minecraft Wiki](https://zh.minecraft.wiki/w/%E5%B1%9E%E6%80%A7?variant=zh-cn)），包含了武器伤害、护甲、水下的氧气呼吸等修正，这里我们添加了Attributes这个已实现好的枚举中的攻击伤害和攻击速度修正，并且采用直接数值相加的操作来进行修正，Operation这个枚举中还有其他修正计算方式，在wiki中同样可以查阅。

我们构建完了这个自定义物品后使用了lang方法，这一方法可以直接给我们提供en_us本地化里的翻译，也即把该物品本地化做我们提供的String；

作为一把手持工具，默认的物品贴图并会贴合正常的工具手持样式，所以我们还需要提供除了item/generated之外的模型：幸好Minecraft原版本身就有很多工具/武器，我们可以直接复用。
model方法需要参数为(DataGenContext, RegistrateItemModelProvider) -> ItemModelBuilder，也就是给我们提供了DataGenContext和RegistrateItemModelProvider，然后要求我们提供ItemModelBuilder.
RegistrateItemModelProvider里对于Minecraft原版就有的特殊模型已经给我们作为函数提供出来，手持工具的ItemModelBuilder可以直接使用handheld方法来，而其参数则是一个NonNullSupplier<T extends ItemLike>
也就是DataGenContext（虽然一般其他地方需要NonNullSupplier<T extends ItemLike>时我们都是提供ItemEntry，但现在我们正是在注册ItemEntry，所以无法直接使用ItemEntry，所以就需要这一参数来实现）

register()方法调用注册物品后，运行gradle任务runData，在我们的src/generated/resources/assets/<modid>/models/item目录下可以找到我们的example_weapon.json，其中parent键名对应的值已经由默认的item/generated变为item/handheld，也就是正常的手持工具模型了。

# 自定义模型

如果我们的物品已经不是原版类型的贴图模型，而是需要一个自定义的模型，那么除了在我们的src/main/resources/assets/<modid>/models/item下自己创建同名的model json外，如果我们还想要利用自动资源数据生成的功能，来给批量物品提供模型生成，我们可以在src/main/resources/assets/<modid>/models/item下创建一个模板model json作为我们的parent（模型定义的相关知识条目可以参阅[Minecraft Wiki](https://zh.minecraft.wiki/w/%E6%A8%A1%E5%9E%8B?variant=zh-cn)），然后使用pvd的parent方法来获取这个model parent：
```java
    public static final ItemEntry<Item> CUSTOM_ITEM =
            REGISTRATE.item("custom_item", Item::new)
                    .model((ctx, pvd) ->
                    {
                        var name = pvd.name(ctx);
                        pvd.getBuilder(name).parent(new ModelFile.UncheckedModelFile(pvd.modLoc("item/custom_weapon"))).texture("layer0", pvd.modLoc("item/example_weapon"));
                    })
                    .register();
```
这里我们直接从pvd获取原始的ItemModelBuilder，然后根据parent，texture等model json的键名提供相应的值，这里的parent需要的是我们放在我们自己手写提供的ModelFile，我们直接new一个ModelFile.UncheckedModelFile给他即可，其中的资源路径是我们在src/main/resources/assets/<modid>/models/目录下的路径去除json文件后缀名。
实际上ModelBuilder可以帮助我们直接在java代码里提供json文件里面写入的键值对，在代码复用时可以节省我们复制粘贴修改json文件。

这里作为Mod开发不再对模型设计做过多详细解释。