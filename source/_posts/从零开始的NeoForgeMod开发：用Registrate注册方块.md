---
title: 从零开始的NeoForgeMod开发：用Registrate注册方块
date: 2025-01-02 13:25:56
tags: [mcmod, dev]
categories: [mcmod]
---

# 注册基本方块

首先我们像前面提到的那样，把方块相关的注册全部分离在一个新的类中，并在类的最末尾添加一个没有内容的静态方法：
```java
    public static void register() {}
```
然后在MOD的入口中执行该register方法，注意放在前面我们的物品注册之后：
```java
        NMBItems.register();
        NMBBlocks.register();
```
正如我们前面所说，由于java的static特性，我们这样调用一下类末尾的static register方法，在register之前的static声明的内容都会被加载，同时我们在Items里最开头声明的
创造模式标签页此时同样也会作为我们方块的默认标签页，而无需我们再手动添加进入。

前面做完了使用Registrate来添加物品，那么现在来添加方块，也是一样的简单：
```java
    public static final BlockEntry<Block> SULPHUR = 
            REGISTRATE.block("sulphur_block", Block::new)
                    .defaultLang()
                    .defaultBlockstate()
                    .simpleItem()
                    .register();
```
同注册简单的物品一样，我们直接调用Registrate的block方法，传入方块名称和构造函数，然后一路default下来最后register注册完。其中：

方块构造函数与Item类似，都是要求由一个Properties作为入参返回一个Block的函数，我们如果想要简单修改Properties也可以直接使用λ：
```java
p -> new Block(p.xxx)
```

defaultLang()与物品注册时的defaultLang一致，都是提供一个默认的本地化。

defaultBlockstate()方法则是提供一个基本的六面均为同一材质无特殊方块状态（Block State）的blockstate json文件及其对应的model json文件。

simpleItem()方法则是自动给该方块添加一个Block Item，非常快捷方便！但需要注意一点，simpleItem默认引用的model名称与我们在block()方法中提供的方块名称一致，
所以在后续我们自定义Blockstate及model的时候需要注意。

最后我们在resources里面添加对应的纹理src/main/resources/assets/nullamultiblock/textures/block/sulphur_block.png，然后运行runData，
Registrate即可为我们自动生成本地化资源和方块状态及模型文件。

如果说在注册物品上仅仅是给我们模型json自动生成方便了一点（毕竟本地化资源我们还要考虑多种语言的本地化，依然需要手工操作），
那在注册方块上Registrate则方便了我们手写非常多且没有代码提示的Blockstate和Model json文件，还免去我们再次注册Block Item的代码，
相比我们直接使用NeoForge的方法来注册，省下了非常多的代码，也不用烦恼Block Item应该放在Items的注册里还是在Blocks的注册的问题！

# 自定义方块状态

方块的模型与方块状态有很大的关系：例如像原版的熔炉这样具有朝向的方块，他就具有一个名为FACING的方块状态，并且根据这个FACING，
Blockstate json会遍历这个状态把他与对应的Model json关联起来。
也就是说，Minecraft世界中的一个方块显示为什么样，首先是由Blockstate来定义他根据不同状态显示何种Model，再由Model来描述他如何渲染。

我们这里创建一个与熔炉类似的方块，首先他有个正面朝向需要由FACING来定义，其次，他还有一个状态LIT来表示当前是否在燃烧中。

## 方块状态与方块的关系

虽然方块状态看上去应该是表示方块内部状态的方块类内的Field，但是我们都知道，在Minecraft中，
我们在世界里放置的方块并不是对应一个new出来的方块类实例，方块类实例在游戏中只存在一个，所以类似的，方块状态也不是存在于方块内部里面的一个可变状态，事实上我们用final修饰
而更像是一种独立的对方块的辅助描述，也类似方块实体（Block Entity）。因此我们声明方块状态的时候，如果是原版已有的方块状态，那么我们可以直接使用原版的方块状态，
其定义于BlockStateProperties里面，比如我们现在使用的FACING和LIT就在其中；如果我们需要创建新的方块状态，那么我们也像我们之前注册方块或物品所作的那样，
建一个新的类来统一存放这些方块状态。

## 声明方块状态

要声明方块状态我们直接在我们的方块类里面加入以下声明：

```java
    public static final DirectionProperty FACING = BlockStateProperties.HORIZONTAL_FACING;
    public static final BooleanProperty LIT = BlockStateProperties.LIT;
```
注意到我们使用了原版已有的方块状态所以不再新建，另外我们的FACING使用的是HORIZONTAL_FACING，这是因为熔炉的正面永远只朝向东西南北四个方向，顶面和底面都是同一材质，
所以使用的是HORIZONTAL_FACING，这个方块状态里面只有东西南北四个方向的枚举，如果使用BlockStateProperties.FACING则还包含了上下总共六个方向枚举，
也就是意味着“正面”还可以朝上和朝下。

正如我们上面介绍方块状态那样所说，其实光声明并不会在游戏里面真的添加状态进去，毕竟这里也没有用到任何注入，不可能让程序自动感知我们的类里有多少方块状态，
所以我们还要进行方块状态的注册：

```java
    public CombustionChamber(Properties properties) {
        super(properties);
        registerDefaultState(getStateDefinition().any()
                .setValue(FACING, Direction.NORTH)
                .setValue(LIT, false));
    }
    
    @Override
    protected void createBlockStateDefinition(StateDefinition.Builder<Block, BlockState> pBuilder) {
        // this is where the properties are actually added to the state
        pBuilder.add(FACING, LIT);
    }
    
    @Override
    public BlockState getStateForPlacement(BlockPlaceContext pContext) {
        return getStateDefinition().any()
                .setValue(FACING, pContext.getHorizontalDirection().getOpposite())
                .setValue(LIT, false);
    }
```

这里我们在“燃烧室方块”的构造函数里面调用了registerDefaultState方法，用来表示缺省的方块状态值；真正把方块状态添加到方块上面的代码是重载方法createBlockStateDefinition：
把我们声明的FACING和LIT添加到入参pBuilder里面即可；重载的另一个方法getStateForPlacement则表示我们在世界里放置方块的时候需要设置为什么方块状态，也因此才能获得符合预期的
方块模型显示：例如：我们设置FACING的方向为
```java
pContext.getHorizontalDirection().getOpposite()
```
这样就是把方向朝向了与玩家朝向相反的方向，这样才能保证燃烧室“正面”朝向玩家。

那给方块添加了状态，我们的Blockstate json和model也要有相应的改变，毕竟我们要做的“燃烧室”与简单的方块不同，有正面、侧面以及顶面三种材质图片，
同时根据LIT的方块状态不同，正面还要显示出不同的材质来表示当前正在燃烧。这里就需要在注册的时候修改defaultBlockstate()转而使用blockstate方法来自定义相关json文件：
```java
    public static final BlockEntry<CombustionChamber> COMBUSTION_CHAMBER = REGISTRATE.block("combustion_chamber", CombustionChamber::new)
            .defaultLang()
            .blockstate((ctx, pvd) -> {
                var unlit = pvd.models().cube("block/combustion_chamber",
                        pvd.modLoc("block/combustion_chamber_top"),
                        pvd.modLoc("block/combustion_chamber_top"),
                        pvd.modLoc("block/combustion_chamber_front"),
                        pvd.modLoc("block/combustion_chamber_side"),
                        pvd.modLoc("block/combustion_chamber_side"),
                        pvd.modLoc("block/combustion_chamber_side"));

                var lit = pvd.models().cube("block/lit_combustion_chamber",
                        pvd.modLoc("block/combustion_chamber_top"),
                        pvd.modLoc("block/combustion_chamber_top"),
                        pvd.modLoc("block/combustion_chamber_front_lit"),
                        pvd.modLoc("block/combustion_chamber_side"),
                        pvd.modLoc("block/combustion_chamber_side"),
                        pvd.modLoc("block/combustion_chamber_side"));

                pvd.horizontalBlock(ctx.get(), blockState -> blockState.getValue(LIT) ? lit : unlit);
            })
            .simpleItem()
            .register();
```

blockstate方法需要传入一个λ，他的入参是DataGenContext<Block, CombustionChamber> ctx和RegistrateBlockstateProvider pvd，然后无需返回值。
这个λ其实是把RegistrateBlockstateProvider直接提供给我们让我们修改来获得我们需要的Blockstate json及model json，同时因为我们还在注册方块的过程中，
但可能还是需要直接引用方块实例，所以还有DataGenContext方便我们获取。

这里起作用的核心代码是
```java
pvd.horizontalBlock(ctx.get(), blockState -> blockState.getValue(LIT) ? lit : unlit);
```
这是RegistrateBlockstateProvider里面定义好的一个方法，专门用来处理像这样有HORIZONTAL_FACING的方块，可以自动生成对应的Blockstate json。
当然我们的方块状态不止HORIZONTAL_FACING一个，还要一个表示当前是否在燃烧的LIT，并且，方块朝向不影响材质，而LIT则影响我们的正面材质
所以我们还要提供两个不同的Model文件unlit和lit：我们使用
```java
pvd.models().cube()
```
来生成一个包含六个面的普通实心方块，注意第一个参数是模型的名字，使用的是资源路径，剩下六个参数就是表示六个面的贴图材质，按顺序分别是下上北南东西面

而方块物品（Block Item）的话我们没有特殊需求（例如原版漏斗和坩埚那样），就依然使用simpleItem直接从一个Cube Block里面生成。这里需要注意一点的是：
simpleItem生成的方块物品自动引用的是与方块同名的方块模型，因此我们在blockstate()方法里面提供的模型里面，默认不燃烧状态的model name与方块同名，而不是跟
燃烧状态的方块模型block/lit_combustion_chamber使用同样的命名方式给命名做block/unlit_combustion_chamber就是这个原因，只有这样我们的方块物品才能找到unlit模型文件。
如果想要完全自定义命名，那么方块物品的生成则需要另外再做，这里显得不必要，所以我们把默认的状态模型名字给命名为与方块同名。

## 方块状态有何用

方块状态的一个基本用处就是储存一些有限的数据：例如像羊毛陶瓦这种方块可以有16色，并且他们都是同种方块只不过颜色表现不同而已，
那么注册16个独立不同的方块，在我们需要引用羊毛或陶瓦而不关心他们颜色时候，就需要同时引用所有16个方块。此时给方块内加入一个方块状态来表示他具有多种颜色，
然后在Blockstate json和Model json里面提供材质变种就方便我们后续在代码中引用了。

方块实体（Block Entity）也可以做到存储数据，但跟方块状态有几个不同，一是：方块状态不仅需要一个json文件来描述，而且我们还是用static final来修饰的，因此方块状态最好用来存储一些
枚举数量有限且枚举起来数量较少的数据（例如布尔值和方向枚举之类的），而方块实体则像我们平常写代码那样可以用来表示非常大量的状态组合；二是：方块状态与方块的渲染显示息息相关，每个状态
其实都对应一个模型，而方块实体则不然，甚至不需要参与渲染。

## horizontalBlock()详解

那么如上所述我们有时候注册方块需要根据方块状态来提供多种模型，但是Registrate的RegistrateBlockstateProvider里面，只有horizontalBlock是带有一个blockState为入参的λ的，
但显然不是所有带方块状态的方块都具有HORIZONTAL_FACING状态，那么这时我们想要更一般地根据方块状态来提供模型，就需要直接来看horizontalBlock方法是如何实现的。

horizontalBlock()方法其实就是调用了pvd.getVariantBuilder(block).forAllStates的方法，forAllStates就可以让我们传入一个λ，来判断方块状态，不过这里需要λ返回的不是ModelFile，
而是ConfiguredModel。而ConfiguredModel也可以很简单地直接从ModelFile生成，直接调用ConfiguredModel.builder().modelFile(xxx).build()即可构建完毕。这里的ConfiguredModel类似pvd提供了
一些自定义的操作xyzuv轴的旋转量，这里涉及模型制作，按下不表。

# 终
至此，我们就可以制作一个简单地仅仅在创造模式物品栏里添加物品和方块的mod了。
