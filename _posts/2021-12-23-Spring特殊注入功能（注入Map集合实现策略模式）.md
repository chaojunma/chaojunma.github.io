---
layout: post
title: "Spring特殊注入功能(注入Map集合实现策略模式)"
date: 2021-12-23
categories: 后端
tags: Spring 设计模式
--- 


Spring提供通过`@Resource`注解将相同类型的对象注入到Map集合，并将对象的名字作为key，对象作为value封装进入Map，下面我们来具体实现一下：

**首先我们定义一个抽象类**

```java
public abstract class TaskAbstractHandler {

    abstract public boolean handleJob(String message);
}
```

**定义多个对象分别继承上面的抽象类**

```java
@Slf4j
@Component("taskA")
public class TaskAHandler extends TaskAbstractHandler {
    @Override
    public boolean handleJob(String message) {
        // TODO 实现taskA具体的业务逻辑
    }
}
```

```java
@Slf4j
@Component("taskB")
public class TaskBHandler extends TaskAbstractHandler {
    @Override
    public boolean handleJob(String message) {
        // TODO 实现taskB具体的业务逻辑
    }
}
```

**注入Map对象**

```java
@Slf4j
@Component
public class ThirdMQListener implements MessageListener {

    @Resource
    private Map<String, TaskAbstractHandler> taskHandlerMap;

    @Override
    public Action consume(Message message, ConsumeContext consumeContext) {

        // 获取消息体
        byte[] body = message.getBody();
        String messageBody = new String(body);

        JSONObject json = JSON.parseObject(messageBody);
        // 获取任务编号
        String taskCode = json.getString("taskCode");

        // 根据tag获取具体调用方
        TaskAbstractHandler taskHandler = taskHandlerMap.get(taskCode);

        if (taskHandler == null) {
            log.error("No object found according to the task code[{}]", taskCode);
            return Action.ReconsumeLater;
        }

        boolean isSuccess = taskHandler.handleJob(messageBody);
        if (isSuccess) {
            return Action.CommitMessage;
        } else {
            return Action.ReconsumeLater;
        }
    }
}
```

上面通过`@Resource`注解将TaskAbstractHandler类型的对象注入到Map集合中，再根据消息体中的任务编号从taskHandlerMap对象或获取到具体的执行任务对象，从而根据任务编号执行不同的策略。