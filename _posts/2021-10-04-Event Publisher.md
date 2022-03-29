---
title: "Spring ApplicationEventPublisher + Async"    
layout: post
read_time: true    
comments: true   
categories: 
 - project  
toc: true    
toc_sticky: true    
toc_label: contents    
description: ApplicationEventPublisher와 Async를 활용한 비동기 이벤트 프로그래밍
last_modified_at: 2021-10-04     
---



쇼핑몰을 개발하면서 이메일을 통해 비로그인으로 상품을 주문한 유저에게 주문정보와 비밀번호 찾기를 요청했을때 비밀번호 초기화 링크를 제공하도록 했는데 Java환경에서  JavaMail 라이브러리를 사용해 메일을 전송하면 3~5초정도간 메일전송 코드에서 블로킹이 발생하는 문제가 있음을 알았습니다.

그리고 주문을 처리하기위한 OrderController혹은 OrderService 그리고 유저 인증을 처리하기위한 UserService 에서 자주 사용되지않는 메일 전송을 위해 메일과 관련된 빈들과 의존성을 맺는것이 별로 보기좋지 않아 이를 분리하기로 결정했는데 이를 위해 ApplicationEventPublisher와 Async를 사용했습니다.



## ApplicationEventPublisher란?

ApplicationEventPublisher는 IoC컨테이너라고 불리는 ApplicationContext가 상속하고있는 인터페이스로 간단하게 이벤트를 발생시키고 처리할 수 있도록 해줍니다.



아래는 간단한 사용 예 입니다.

먼저 Event객체를 생성합니다.

```java
@Getter
public class MyEvent extends ApplicationEvent {
    private String message;

    public MyEvent(Object source, String message) {
        super(source);
        this.message = message;
    }
}
```

ApplicationEventPublisher에 의해 발생한 이벤트를 수신할 Listener객체를 생성합니다.

```java
@Component
public class MyEventListener implements ApplicationListener<MyEvent> {
    
    @Override
    public void onApplicationEvent(MyEvent myEvent) {
        System.out.println("Received event : " + myEvent.getMessage());    
    }
}
```

위 방식은 Spring 4.2 이전의 방식을 사용한 것이고 Spring 4.2 이후부터는 Event객체에 ApplicationEvent를 상속받지 않아도 되고 Listener를 아래처럼 정의할 수 있게 되었습니다.

```java
@Component
public class MyEventListener {

    @EventListener
    public void onApplicationEvent(MyEvent myEvent) {
        System.out.println("Received event : " + myEvent.getMessage());
    }
}
```

Event객체와 Listener를 만든 후 아래처럼 ApplicationEventPublisher를 사용해 이벤트를 발생시킬 수 있습니다.

```java
@Autowired
ApplicationEventPublisher applicationEventPublisher;

public void publish() {
    applicationEventPublisher.publishEvent(new MyEvent(this, "Hello"));
}
```

 

## Async란?

Async란 메소드의 호출 후 응답을 기다리지 않고 다음 명령을 수행하는 비동기 프로그래밍을 말합니다.

스프링에서는 이러한 비동기 프로그래밍을 쉽게 적용할 수 있도록 @EnableAsync, @Async등의 어노테이션을 제공합니다.

@Async어노테이션을 사용해 쓰레드를 생성하면 ThreadPoolTaskExecutor에 의해 동작하게되는데 다음과 같은 설정을 통해 별도로 쓰레드풀을 빈으로 관리할수도 있습니다.

```java
@EnableAsync
@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("executor-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        executor.initialize();
        return executor;
    }
}
```

이제 어떠한 로직을 비동기적으로 수행시키고자 할때 메소드에 @Async 어노테이션을 적어주기만하면 해당 메소드가 호출될 때 쓰레드풀에서 사용되고있지 않은 쓰레드를 가져와 기존 쓰레드와 다른 쓰레드에서 메소드를 동작시키게 됩니다.

```java
@Async
public void handler() {
    ...
}
```



## Event와 Async

발생된 Event를 전달받은 EventListener는 비동기적으로 동작될거같지만 실제로는 동기적으로 동작합니다.

물론 리스너가 동기적으로 동작해도 상관없는 경우가 있겠지만 저는 이메일 전송이 완료되기까지 3~5초 동안 응답을 전송하지못하고 대기하는 문제를 해결해야하기 때문에 리스너가 비동기적으로 동작하도록 만들어야 합니다.

리스너가 비동기적으로 동작하도록 하려면 다음과 같이 @EventListener가 붙어있는 메소드에 @Async어노테이션을 붙여주기만하면 됩니다.

```java
@Component
public class NameEventListener {

    @EventListener
    @Async
    public void listener(NameEvent nameEvent) throws InterruptedException {
        Thread.sleep(3000);
        System.out.println("Hello My name is " + nameEvent.getName());
    }
}
```

/event/publish 라는 path로 요청을 받을때 위 이벤트를 발생시키는 코드를 작성하고 Postman을 사용해 테스트를 해봤습니다.

결과는 아래처럼 사용자는 이벤트 처리가 끝날때까지 기다리지 않고 먼저 응답을 받고 서버에서 해당 이벤트 처리를 비동기적으로 수행하는것을 볼 수 있습니다.

![1](/assets/image/event_publisher/1.gif)

