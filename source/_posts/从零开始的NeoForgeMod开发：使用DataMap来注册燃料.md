---
title: 从零开始的NeoForgeMod开发：使用DataMap来注册燃料
date: 2025-01-07 15:42:56
tags: [mcmod, dev]
categories: [mcmod]
---

# DataMap介绍

前面我们创建了一个类似原版熔炉的燃烧室，那么燃烧室显然也需要燃料，那我们如何定义一个有燃烧温度和燃烧时间的燃料呢？原版熔炉使用的是DataMap机制。

DataMap也是资源（Resource）的一种，他也是以JSON的形式存储，形成物品-数据的映射，但是与配方（Recipe）不同，他是以一种物品为key来读取数据，而配方则是匹配一个物品输入模板，
因此配方相比DataMap而言定义起来也更复杂。
在“燃料”这一类型上，我们只需要使用DataMap即可。

# 定义DataMap

## 定义数据

我们这里需要两个基本量：燃烧温度和燃烧时间，燃烧温度我们直接用enum来表示低温、中温和高温：
```java
public enum HeatLevel {
    NONE,
    LOW,
    MEDIUM,
    HIGH,
    ;
}
```

而燃烧时间我们直接使用int来表示，单位为刻（tick），最终我们的燃料数据是：
```java
public record FuelData(HeatLevel heatLevel, int burningTime)
```

由于我们需要把数据按JSON的形式存储，所以我们还需要一个CODEC来转换为JSON：

```java

public record FuelData(HeatLevel heatLevel, int burningTime) {
    public static final Codec<FuelData> CODEC = RecordCodecBuilder.create(instance -> instance.group(
            Codec.INT.fieldOf("heat_level").forGetter(data -> data.heatLevel.ordinal()),
            Codec.INT.fieldOf("burning_time").forGetter(FuelData::burningTime)
    ).apply(instance, (heatLevel, burningTime) -> new FuelData(HeatLevel.values()[heatLevel], burningTime)));
}
```

注意到由于Minecraft的CODEC并没有提供enum的转换，所以我们需要把enum转换成int再使用CODEC的INT系列方法。

## 注册DataMap

接下来我们要注册DataMap，根据Minecraft的系列注册流程，我们还要先注册一个DataMapType，并且后续使用我们自定义DataMap的时候，也是引用这一个Type，
所以我们也可以按照注册方块和物品的方法，把DataMapType统一放到一个类里面：
```java
public class DataMapTypes {
    public static final DataMapType<Item, FuelData> FUEL_DATA = DataMapType.builder(
            ResourceLocation.fromNamespaceAndPath("nullamultiblock", "fuel_data"),
            // The registry to register the data map for.
            Registries.ITEM,
            // The codec of the data map entries.
            FuelData.CODEC
    ).build();
}
```

这里我们注册DataMap为物品到燃料数据的映射，builder需要的参数为资源路径，物品注册表和我们燃料数据的CODEC。

那最后我们注册这一DataMapType，还需要使用注册事件：

```java
public static void registerDataMapTypes(RegisterDataMapTypesEvent event) {
    event.register(DataMapTypes.FUEL_DATA);
}
```
将这一方法在MOD的入口中加入调用即可实现注册；

## 使用Registrate来手动生成数据

上面我们有讲到：DataMap也是资源，同样按JSON文件存储，我们当然可以手动创建对应的JSON文件来手写提供，但手写JSON数据不仅格式麻烦，名字易错，也不好debug。
那我们可以直接使用Registrate来进行数据生成：

```java
public static void provideDataMaps() {
    REGISTRATE.addDataGenerator(ProviderType.DATA_MAP, pvd -> pvd.builder(DataMapTypes.FUEL_DATA)
            .add(ItemTags.LOGS, new FuelData(HeatLevel.LOW, 600), false)
            .add(ItemTags.PLANKS, new FuelData(HeatLevel.LOW, 300), false)
            .add(ItemTags.COALS, new FuelData(HeatLevel.MEDIUM, 1600), false));
}
```

我们这里调用REGISTRATE的addDataGenerator方法，DataGenerator是NeoForge提供的机制，我们这里使用Registrate包装后的方法：
ProviderType是Registrate自定义好的一系列用于数据生成的类型，我们这里要生成的是DataMap，pvd需要的DataMapType就是我们上面定义的
最后直接调用.add方法，传入ItemTag和对应的燃料数据FuelData就可以自动生成JSON条目，注意最后的布尔参数，表示的是是否覆盖已有的数据
这里我们一般全都选择false，即不覆盖已有数据，防止意外修改了原版或者其他mod的数据资源

最后我们还需要在MOD入口函数处调用该静态方法，即可在runData task中自动生成对应的燃料数据。