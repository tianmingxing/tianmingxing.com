---
title: 使用MapStruct框架实现Bean之间属性映射
date: 2019-12-04 23:07:00
tags:
- MapStruct
- Bean属性映射
categories:
- 开源框架
---

![](/images/mapstruct.png)

在实际项目开发过程中我们经常会遇到把A类的属性复制到B类中的这种场景，比如实体类DO对象从数据库查询出来，在给到调用方之前很可能需要转成DTO传输对象，那这个时候就需要将DO对象的字段值复制到DTO对象中，如果字段比较少可能你手写就行了，但如果要复制的字段比较多，超过50个，并且这种场景在项目中出现频繁，该怎么办？仍然还是去手写，然后一点一点复杂吗？如果这样做那效率必然是低下的，而且没有一点儿技术含金量，花大把时间写这种样本代码对自己成长帮助不大。接下来介绍的是一款让你更优雅的实现Bean之间属性映射的框架，并且在同类框架中性能是最好的。

# 介绍

[MapStruct](https://mapstruct.org/)是一个代码生成器，它基于约定优于配置的方法极大地简化了Java bean类型之间映射的实现。

通过上面的介绍我们应该能够理解到这么几点，首先它是一个代码生成器，就是用来帮开发者自动生成代码的工具，只需要通过简单的代码就可以实现原来手工编写的样板代码，因为它采用约定大于配置的设计思想，所以开发者只需要掌握简单的代码编写就可以了。

也就是说人家框架帮你自动生成了原先手工编写的代码，但实际上那些手工编写的代码还是存在的，只不过你没有编写，框架帮你自动生成了而已。这其实也回到框架的本质，事情还是那些事，就看你来做，还是它来做，它如果多做，你就少做，甚至可以不做。这里提到的**它**指的是各种框架，它的本质就是帮开发者做了一些事情。

## 特点

与动态映射框架相比，MapStruct具有以下优点：

* 通过使用普通方法调用而不是反射来快速执行
* 速度快：由于MapStruct不采用所谓的反射机制，而是像开发者原来手工逐个赋值那样编码，所以没有额外的性能损失，跟你自己写的代码是一样的。
* 编译时类型安全性
* 展示生成报告：在生成代码过程中如果发现映射不完整、不正确会立即输出日志。

## 集成

介绍如何在项目中集成进来。

### 工作原理

在做这件事情之前我们要搞清楚它的运行原理，它到底是怎么生成代码的？

1. 在代码编译时会触发MapStruct插件运行
2. 当MapStruct运起来之后会扫描它自己特定注解的类
3. 解析类中的方法按照自己的策略在项目编译目录（build）下生成实现类，如果生成过程中出现异常则会输出日志，并中断当前整个项目编译工作。

### 在项目中集成MapStruct框架

在项目中添加MapStruct的依赖，并且配置MapStruct编译插件，让程序在编译时触发MapStruct插件工作，否则代码肯定不会生成，那在客户端调用时就会出错。

* 如果是maven项目，在pom文件中添加下面的代码：

```xml
<properties>
    <org.mapstruct.version>1.3.1.Final</org.mapstruct.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.5.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${org.mapstruct.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

* 如果是gradle项目，在build文件中添加下面的代码：

```groovy
dependencies {
    implementation "org.mapstruct:mapstruct:${mapstructVersion}"
    annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"

    // If you are using mapstruct in test code
    testAnnotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
}
```

# 使用示例

如果我们需要将Car对象的属性赋值给CarDto对象，那可以按照下面方法做：

* 定义一个接口，如下所示就可以了。

```java
@Mapper
public interface CarMapper {
    @Mapping(source = "make", target = "manufacturer")
    @Mapping(source = "numberOfSeats", target = "seatCount")
    CarDto carToCarDto(Car car);
}
```

基于上面的约定MapStruct将自动生成下面的代码，看起来是不是跟自己以前手工编写的差不多？

```java
// GENERATED CODE
public class CarMapperImpl implements CarMapper {

    @Override
    public CarDto carToCarDto(Car car) {
        if ( car == null ) {
            return null;
        }

        CarDto carDto = new CarDto();

        if ( car.getFeatures() != null ) {
            carDto.setFeatures( new ArrayList<String>( car.getFeatures() ) );
        }
        carDto.setManufacturer( car.getMake() );
        carDto.setSeatCount( car.getNumberOfSeats() );
        carDto.setDriver( personToPersonDto( car.getDriver() ) );
        carDto.setPrice( String.valueOf( car.getPrice() ) );
        if ( car.getCategory() != null ) {
            carDto.setCategory( car.getCategory().toString() );
        }
        carDto.setEngine( engineToEngineDto( car.getEngine() ) );

        return carDto;
    }

    private EngineDto engineToEngineDto(Engine engine) {
        if ( engine == null ) {
            return null;
        }

        EngineDto engineDto = new EngineDto();

        engineDto.setHorsePower(engine.getHorsePower());
        engineDto.setFuel(engine.getFuel());

        return engineDto;
    }
}
```

看到这里你应该明白了，通过简单的接口定义就可以不用写那么多样板代码，真是太爽了。MapStruct给开发者的选择很灵活，你不用非得用接口来向它告诉你的**约定**，你可以用抽象类，或者就是普通的类，这些都可以，具体可以看看官方文档。

# 主要功能介绍

由于官方文档实在时写的太详细了，本文实在没有理由赘述一遍， 所以在此把MapStruct能够实现的功能特性列举出来，让大家在遇到类似问题时知道它可以有方案解决。

说明：源（source）对象指的是从哪个Bean中复制数据，目标（target）对象指的是要把源对象的数据复制到哪个Bean对象中去。可以理解Bean之间映射就是一个从哪到哪的过程。

1. 支持maven/gradle/ant工具所构建的项目
2. 在某些情况下，可能需要手动实现从一种类型到另一种类型的特定映射，而MapStruct无法生成这种映射，你可以向映射器中添加自定义方法。
3. 可以用多个源对象的数据映射到一个对象上，也就是说可以同时把A和B的数据复制到C对象中。
4. 可以通过引用更新目标对象，默认在复制时会生成新的对象并返回，但有时候可能不需要生成新的对象，只希望它在既有对象上进行复制。
5. Bean的字段如果没有提供getter/setter方法也可以进行复制，它会通过实例直接访问属性来达到目的。
6. 如果Bean提供了自己的工厂，即通过Builder构造自己，也可以被识别到。
7. 获取映射器之后客户端才能调用，获取的方式支持Mappers工厂、CDI依赖注入和Spring依赖注入。
8. 类型隐式转换
9. 嵌套对象自动映射
10. 在当前映射器中可以调用其他映射器，可以是自定义映射器。
11. 集合映射
12. 流映射
13. 枚举类映射
14. 映射字段控制：常量，默认值，忽略字段，NULL值检查和处理策略，异常处理
15. 映射配置共享、继承和反向映射
16. 装饰器映射，也就是在映射前后做一些自定义操作。
17. 提供SPI接口，可以修改框架的部分实现。

----
参考文献：
1. https://mapstruct.org/documentation/stable/reference/html





