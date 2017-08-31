---
layout: default
title:  "Hibernate validator校验"
date:   2017-08-31 16:16:16
categories: Hibernate validator
---
# 服务端校验使用说明
---
服务器端校验一直是各个系统的薄弱环节，许多系统甚至没有服务器端的校验。Hibernate Validator实现了JavaEE的注解校验，同时对其进行了扩展，通过注解的方式实现规则校验，同时还可以自定义校验规则，十分方便。
<!--more-->

## 一、校验使用方式

### 1、默认校验
> 开发人员只需要在接口请求参数需要校验的属性、方法、或类上添加相应的注解约束条件，框架将会自动进行校验，并在校验失败后直接封装错误信息返回给用户，开发者不需要关心校验不通过的处理流程。

以包机切位库存接口为例：接口方法为
```java
public ResponseObj<StockOperateResponse> stockFreeze(HttpServletRequest request, HttpServletResponse response, StockOperateRequest params) throws Exception {
      ......
}
```
请求参数为StockOperateRequest，如校验用户office号不能为空：
```java
public class StockOperateRequest {

  private Long id;

  @NotNull
  private String officeNo;

  private String orderNo;
}
```

### 2、分组校验
> 如果有两个方法都使用了相同的参数，那么在两个方法中都会对参数进行校验。但是比如新增和修改操作，在新增操作中并不需要校验id，因为id是数据库生成并返回的；而在修改操作则id必须校验，否则不知道用户修改的哪条数据。这种使用共同参数，但是各自又采用不同校验方案时可采用分组校验。

①定义一个分组类FirstGroup：
```java
public interface FirstGroup {
}
```
②指定id的校验分组为FirstGroup：
```java
public class StockOperateRequest {

  @NotNull(groups={FirstGroup.class})
  private Long id;

  @NotNull
  private String officeNo;

  private String orderNo;
}
```
③由于新增方法add()不需要校验id，因此新增方法不做任何改变。更新方法update()需要校验id，使用注解@Validated指定参数使用FirstGroup分组进行校验：
```java
public ResponseObj<StockOperateResponse> update(HttpServletRequest request, HttpServletResponse response, @Validated({FirstGroup.class}) StockOperateRequest params) throws Exception {
      ......
}
```
**特别说明**：框架限定在任何情况下都会对默认分组（即未传入groups参数的注解）进行校验。如上面的add()将校验默认分组，update将校验FirstGroup分组和默认分组。

## 二、自定义注解
> 在hibernate和javaEE中已经定义了以下比较通用的注解：
> * AssertFalse
> * AssertTrue
> * DecimalMax
> * DecimalMin
> * DecimalMin
> * Future
> * Max
> * Min
> * NotNull
> * Null
> * Past
> * Pattern
> * Size
> * Email
> * Length
> * NotBlank
> * NotEmpty
> * Range
> * URL
> * CreditCardNumber
> * ScriptAssert  
>
>然而在实际应用中，这些注解还不能完全满足需求，因而需要根据业务场景自定义注解，比如参数必须在指定日期之后。  

### 1、自定义注解
①定义注解类
定义@AfterData类如下：
```java
@Constraint(validatedBy = AfterDataValidator.class) //注解实现类
@Target( { java.lang.annotation.ElementType.METHOD, java.lang.annotation.ElementType.FIELD })   //注解作用范围
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)  
@Documented  
public @interface AfterData {
  String message() default "指定日期必须晚于{predate}";
  String predate() default ;

  //下面这两个属性必须添加  
  Class<?>[] groups() default {};
  Class<? extends Payload>[] payload() default {};
}
```
②定义注解实现类AfterDataValidator，需实现接口ConstraintValidator的isValide()方法：
```java
public class AfterDataValidator implements ConstraintValidator<AfterData, String>{
	String date;

	@Override
	public void initialize(AfterData arg0) {
		this.date = arg0.predate();
	}

	@Override
	public boolean isValid(String arg, ConstraintValidatorContext constraintContext) {
		System.out.println("arg:" + arg);
		System.out.println("predate:" + date);
		if(.equals(date)){
  		constraintContext.disableDefaultConstraintViolation();
          constraintContext.buildConstraintViolationWithTemplate("没有比较的前置日期").addConstraintViolation();
		  return false;
		} else {
			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
			try {
				if(sdf.parse(date).before(sdf.parse(arg))){
					return true;
				}else{
					return false;
				}
			} catch (ParseException e) {
				constraintContext.disableDefaultConstraintViolation();  
	            constraintContext.buildConstraintViolationWithTemplate("传入的日期格式有误").addConstraintViolation();
				return false;
			}
		}
	}
}
```

### 2、定义组合校验
系统中用户密码必须满足以下条件：
* 不能为空(@NotNull)
* 长度不能小于8位(@Length(min=8))
* 必须包含数字、大写字母、小写字母及特殊字符(假定已自定义注解@MyAnnotation实现此功能校验)  

那么是否在每个用到password的地方都必须有以下代码：
```java
@NotNull
@Length(min=8)
@MyAnnotation
private String password;
```
对于这种情况，可以定义一个组合校验@AptPassword:
```java
@NotNull
@Length(min=8)
@MyAnnotation
@Constraint(validatedBy = {})
@Target({ java.lang.annotation.ElementType.METHOD, java.lang.annotation.ElementType.FIELD })
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Documented
public @interface AptPassword {
	String message() default "密码不满足要求";

	Class<?>[] groups() default {};
	Class<? extends Payload>[] payload() default {};
}
```
然后在password字段使用注解@AptPassword即可：
```java
@AptPassword
private String password;
```

## 三、多属性校验
> 由于自定义注解isValid()实现方法传入的参数是注解所在的Field或注解方法对应的Field，所以无法获取到此Field所属对象的其他Field，故使用Field和Method注解无法实现多个Field之间的关系校验。因此多个Field之间的关系校验需要使用类注解来实现（即指定Target为java.lang.annotation.ElementType.TYPE）。

如：有政策对象如下，需校验生效日期必须在失效日期之前：
```java
public class PolicyPo{
    private String effectiveDate;//生效日期
    private String expiryDate;//失效日期
}
```
定义注解@DateOrder:
```java
@Constraint(validatedBy = DateOrderValidator.class) //具体的实现  
@Target({ java.lang.annotation.ElementType.TYPE })
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Documented
public @interface DateOrder {
	String message() default "{before}必须早于{after}";
	String before() default ;
	String after() default ;
	Class<?>[] groups() default {};
	Class<? extends Payload>[] payload() default {};
}
```
然后实现类DateOrderValidator：
```java
public class DateOrderValidator implements ConstraintValidator<DateOrder, Object> {
	String before;
	String after;

	@Override
	public void initialize(DateOrder constraintAnnotation) {
		this.before = constraintAnnotation.before();
		this.after = constraintAnnotation.after();
	}

	@Override
	public boolean isValid(Object value, ConstraintValidatorContext context) {
		Class<?> cla = value.getClass();
		try {
			/* 根据field获取对应的get方法 */
			Method methodBefore = cla.getDeclaredMethod("get" + firstToUpcase(before));
			Method methodAfter = cla.getDeclaredMethod("get" + firstToUpcase(after));
			/* 调用get方法获取对应的值 */
			Object resultBefore = methodBefore.invoke(value);
			Object resultAfter = methodAfter.invoke(value);

			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
			/* 判定两日期的先后顺序 */
			return sdf.parse((String) resultBefore).before(sdf.parse((String) resultAfter));
		} catch (Exception e) {
			e.printStackTrace();
		}
		return false;
	}

	/**
	 * 字符串首字母转大写
	 */
	private String firstToUpcase(String str) {
		return String.valueOf(str.charAt(0)).toUpperCase() + str.substring(1);
	}
}
```
最后在政策类上添加注解@DateOrder即可：
```java
@DateOrder(before="effectiveDate", after="expiryDate")
public class PolicyPo{
    private String effectiveDate;//生效日期
    private String expiryDate;//失效日期
}
```
