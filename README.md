# Lombok @Data @Builder 一起使用问题
# 背景
@Data 和 @Builder 是我们使用 lombok 过程中常用的两个注解,@Data可以帮我们将一个普通类变成JavaBean，并自动生成相应属性的 Getter,Setter等方法；而@Builder注解则是应用的构造者模式，通过链式方法帮助我们生成对象。
# 问题
有时候我们会将@Data 和 @Builder这两个注解混合使用，对应的场景为在生成对象的时候通过构造者模式生成对象，然后将生成的对象作为 JavaBean 使用。然而在实际使用时会发现我们将这两个注解放在一起会编译失败。这里我们暂且不评价这样使用的好坏，而是先专注于解决问题。

这个问题已经在新版的 lombok 中解决，你可以通过升级 lombok 为最新版的解决。
下面介绍的方法都是旧版的解决方案。

# 解决方案一：通过添加@NoArgsConstructor 和 @AllArgsConstructor 这两个注解
如下:

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    /**
     * 姓名
     */
    private String name;
}

这种方式会通过@AllArgsConstructor生成一个访问级别为 public 的全参构造器。@Builder可以用生成的全参构造器来构造对象,不再与@Data所包含的@RequiredArgsConstructor发生冲突

# 解决方案二:通过自己重写无参构造器,并添加@Tolerate注解
如下:

@Data
@Builder
@AllArgsConstructor
public class UserDTO {
	@Tolerate
	public UserDTO(){
	}
    /**
     * 姓名
     */
    private String name;
}

这种方式下，自己重写的无参构造器可以帮助我们解决@Data与 @Builder 注解的冲突。

现象分析
为什么会出现 @Data 和 @Builder 混用的问题，以及两者混用导致的问题，这里从我个人的感觉在这里分析一波。

首先需要明确@Data 和 @Builder 在开发中的地位，@Data 的出现很明确，它为一个 Java类赋予了 JavaBean 的全部功能.

在为一个 JavaBean 在增加 @Data注解后，可以使用set方法设置对象属性并且使用get方法读取属性。

// User.java
@Data
public class User{
    /**
     * 姓名
     */
    private String name;
    /**
     * 国籍
     */
    private CountryEnum country;
}
/*-----------------*/
final User user = new User();
// 设置对象属性，setName 方法由 lombok自动生成
user.setName("zhangsan");
user.setCountry(CountryEnum.china);
System.out.println(user);

而 @Builder 方法则不同，构造者模式的初衷是构建并产出复杂产品，在配置好属性后产出的结果是一个产品，此时对于开发者而言更加注重的是使用产品的功能而非获取产品内部的构造。与 JavaBean的区别在于在获取产品后是无法再对产品设置属性亦或者读取属性。而 @Builder 会为类创建一个构造内部类，通过该类可以构造产品。该种情形下，开发者不应该通过 new创建对象而是统一通过构造类创建对象，@Builder 获取到的对象也没有set和get功能。

@Builder
public class Radio{
	private Float band;
	public String listen(){
		return "here now are listening"+band+"frequency";
	}
}
/**************************/
Radio radio = Radio.builder().band(108.2).build();
// 强调对行为的操作而非属性的操作
System.out.println(radio.listen());

那么将 @Builder 与 @Data 合并在一起会获得什么效果呢？从结果上来看，是通过构造器产生的产品是一种特殊的产品，我们可以对"产品"的属性随意进行操作，可以通过 get 和 set方法对其进行操作，虽然这样会改变产品的行为…

对于JavaBean逻辑而言,如果只使用@Builder+@Value 注解会导致一系列反序列化问题,如 FastJson 的解析逻辑中需要一个公有的无参构造器,而 @Value会为当前对象生成一个全参构造器,无参构造器的缺失导致FastJson 在实例化对象的时候发生异常。

@Value
@Builder
public class UserDTO {
    String name;
    Integer age;
}
public static void main(String[] args){
    String str = "{'name':'zhangsan','age':23}";
    UserDTO user = JSON.parseObject(str, UserDTO.class);
    System.out.println(user);
}

Exception in thread "main" com.alibaba.fastjson.JSONException: default constructor not found. class org.example.dto.UserDTO
	at com.alibaba.fastjson.util.JavaBeanInfo.build(JavaBeanInfo.java:574)
	at com.alibaba.fastjson.parser.ParserConfig.createJavaBeanDeserializer(ParserConfig.java:993)
1
2
3
FastJson2 没有这个问题，有感兴趣的同学可以去试试。

两个注解一起使用就好像是两个不同的东西组合在了一起。这样做的效果有什么问题呢？首先@Data注解标记了我们最终的产出注定是一个 JavaBean，那么对于 JavaBean最好不要存在向上面radio那样基于配置而产生的listen行为，因为由于 JavaBean 中的属性是可以设置的，这样有可能导致listen之类的行为得到非预期的效果。

下面来看这样做的好处，构造器模式的使用可以帮助我们通过链式调用的方式快速构建 JavaBean，以往我们需要通过多行set方法实现的JavaBean在构造器模式下只需要一行即可完成，并且简单明了。

// JavaBean 模式下的构造操作
User user = new User();
user.setName("zhangsan");
user.setCountry(CountryEnum.china);
System.out.println(user);
// Builder 模式下的构造操作
User user = User.builder().name("zhangsan").country(CountryEnum.china).build();
System.out.println(user);
