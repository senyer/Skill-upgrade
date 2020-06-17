## MapStruct的作用

类似于Apache的Beanutil.copyProperties的用法，用于对象之间的属性映射。而MapStruct是最高效，性能最好的一款工具。

## 基本用法

### 编写转换源到目标的映射

``` java
 import cn.felord.mapstruct.entity.Car;
 import cn.felord.mapstruct.entity.CarDTO;
 import org.mapstruct.Mapper;
 import org.mapstruct.Mapping;
 import org.mapstruct.factory.Mappers;
 
 /**
  * CarMapping
  *
  */
 @Mapper
 public interface CarMapping {
     /**
      * 用来调用实例 实际开发中可使用注入Spring  不写
      */
     CarMapping CAR_MAPPING = Mappers.getMapper(CarMapping.class);
 
 
     /**
      *  源类型 目标类型 成员变量相同类型 相同变量名 不用写{@link org.mapstruct.Mapping}来映射
      *
      * @param car the car
      * @return the car dto
      */
     @Mapping(target = "type", source = "carType.type")
     @Mapping(target = "seatCount", source = "numberOfSeats")
     CarDTO carToCarDTO(Car car);
 
 }
```

### 步骤讲解

1. 声明一个映射借口：用`@org.mapstruct.Mapper`标记，
2. `@org.mapstruct.Mapping`注解用来声明成员属性的映射：重要属性：source & target。 
3. 【高级用法】 设置转换默认值和常量，当目标值为null的时候，可以设置其默认值，例如：`@Mapping(target = "stringProperty", source = "stringProp", defaultValue = "undefined")` 
4. 对于常量的正确用法(常量不能指定source属性):`@Mapping(target = "stringConstant", constant = "Constant Value")`

> MapStruct最终调用的就是setter和getter方法，不是基于反射实现的，所以性能比较好。

### MapStruct进阶操作

#### 格式化操作

格式化是常用操作，比如：数字格式化、日期格式化

数字格式化：

> `@Mapping(source = "price", numberFormat = "$#.00")`

日期格式化：    如下：将日期集合映射到日期字符串集合

> ```
> @IterableMapping(dateFormat = "dd.MM.yyyy")
> List<String> stringListToDateList(List<Date> dates);
> ```

#### 使用Java表达式

首先在`@org.mapstruct.Mapper` 的 `imports` 属性中导入 `LocalDateTime`，该属性是数组意味着你可以根据需要导入更多的处理类：

```java
@Mapper(imports = {LocalDateTime.class})
```

接下来只需要在对应的方法上添加注解`@org.mapstruct.Mapping` ，其属性`expression` 接收一个 `java()` 包括的表达式：

* 无入参版本：

```java
@Mapping(target = "addTime", expression = "java(LocalDateTime.now())")
```

* 携带入参的版本, 我们将 `Car` 的出厂日期字符串`manufactureDateStr` 注入到 `CarDTO` 的 `LocalDateTime` 类型属性`addTime` 中去：

```java
 @Mapping(target = "addTime", expression = "java(LocalDateTime.parse(car.manufactureDateStr))")
 CarDTO carToCarDTO(Car car);
```

### MapStruct转换Mapper注入到Spring Ioc他容器

如果使用要把Mapper 注入Spring IoC 容器我们只需要这么声明，不用再构建一个单例，就可以像其他 spring bean一样对CarMapping 进行引用了：

``` java
  import cn.felord.mapstruct.entity.Car;
  import cn.felord.mapstruct.entity.CarDTO;
  import org.mapstruct.Mapper;
  import org.mapstruct.Mapping;
  import org.mapstruct.factory.Mappers;
 
 /**
  * CarMapping 注入spring 写法
  *
  */
 @Mapper(componentModel = "spring")
 public interface CarMapping {
     /**
      * 用来调用实例 实际开发中可使用注入Spring  不写
      */
 //    CarMapping CAR_MAPPING = Mappers.getMapper(CarMapping.class);
 
 
     /**
      * 源类型 目标类型 成员变量相同类型 相同变量名 不用写{@link Mapping}来映射
      *
      * @param car the car
      * @return the car dto
      */
     @Mapping(target = "type", source = "carType.type")
     @Mapping(target = "seatCount", source = "numberOfSeats")
     CarDTO carToCarDTO(Car car);
 
 }
```

### 自定义属性类型转换方法

一般常用的类型字段转换 MapStruct都能替我们完成，但是有一些是我们自定义的对象类型，MapStruct就不能进行字段转换，这就需要我们编写对应的类型转换方法，笔者使用的是JDK8，支持接口中的默认方法，可以直接在转换器中添加自定义类型转换方法。

示例中User对象的config属性是一个JSON字符串，UserVo对象中是List类型的，这需要实现JSON字符串与对象的互转。

``` java
default List<UserConfig> strConfigToListUserConfig(String config) {
  return JSON.parseArray(config, UserConfig.class);
}

default String listUserConfigToStrConfig(List<UserConfig> list) {
  return JSON.toJSONString(list);
}
```





### 基于泛型可以写一个基类

``` java
@MapperConfig
public interface BaseMapping<SOURCE, TARGET> {

    /**
     * 映射同名属性
     */
    @Mapping(target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    TARGET sourceToTarget(SOURCE var1);

    /**
     * 反向，映射同名属性
     */
    @InheritInverseConfiguration(name = "sourceToTarget")
    SOURCE targetToSource(TARGET var1);

    /**
     * 映射同名属性，集合形式
     */
    @InheritConfiguration(name = "sourceToTarget")
    List<TARGET> sourceToTarget(List<SOURCE> var1);

    /**
     * 反向，映射同名属性，集合形式
     */
    @InheritConfiguration(name = "targetToSource")
    List<SOURCE> targetToSource(List<TARGET> var1);

    /**
     * 映射同名属性，集合流形式
     */
    List<TARGET> sourceToTarget(Stream<SOURCE> stream);

    /**
     * 反向，映射同名属性，集合流形式
     */
    List<SOURCE> targetToSource(Stream<TARGET> stream);
}
```



### 常见问题

1. 当两个对象属性不一致时，比如User对象中某个字段不存在与UserVo当中时，在编译时会有警告提示，可以在@Mapping中配置 ignore = true，当字段较多时，可以直接在@Mapper中设置unmappedTargetPolicy属性或者unmappedSourcePolicy属性为 ReportingPolicy.IGNORE即可。
2. 如果项目中也同时使用到了 Lombok，一定要注意 Lombok的版本要等于或者高于**1.18.10**，否则会有编译不通过的情况发生。
