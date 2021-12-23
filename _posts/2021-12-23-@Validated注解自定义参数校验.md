---
layout: post
title: "@Validated注解自定义参数校验"
date: 2021-12-23
categories: 分布式
tags: SpringBoot
--- 

参数校验是我们程序开发中必不可少的过程。用户在前端页面上填写表单时，前端js程序会校验参数的合法性，当数据到了后端，为了防止恶意操作，保持程序的健壮性，后端同样需要对数据进行校验。后端参数校验最简单的做法是直接在业务方法里面进行判断，当判断成功之后再继续往下执行。但这样带给我们的是代码的耦合，冗余。当我们多个地方需要校验时，我们就需要在每一个地方调用校验程序,导致代码很冗余，且不美观。


那么如何优雅的对参数进行校验呢？JSR303就是为了解决这个问题出现的，本篇文章主要是介绍Hibernate Validator如何自定义参数校验注解。

**自定义校验注解**

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Constraint(validatedBy = IntValueRangeValidator.class)
public @interface IntValueRange {

    int[] values();

    String message() default "不在取值范围之内";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

**校验详细实现**

```java
public class IntValueRangeValidator implements ConstraintValidator<IntValueRange, Object> {

    private int[] values;

    @Override
    public void initialize(IntValueRange intValueRange) {
        this.values = intValueRange.values();
    }

    @Override
    public boolean isValid(Object o, ConstraintValidatorContext constraintValidatorContext) {

        if (ObjectUtils.isEmpty(o)) {
            return true;
        }

        for (int value : values) {
            if(Objects.equals(o, value)) {
                return true;
            }
        }

        return false;
    }
}
```

**参数校验**

```java
@IntValueRange(values = {1,2}, message = "性别只能为1或2")
private Integer sex;
```