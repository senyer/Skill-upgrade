> ApplicationContext 通过 ApplicationEvent 类和 ApplicationListener 接口进行事件处理。 如果将实现 ApplicationListener 接口的 bean 注入到上下文中，则每次使用 ApplicationContext 发布 ApplicationEvent 时，都会通知该 bean。本质上，这是标准的观察者设计模式。

### 事件发布者

>  注意：再Spring中，服务必须交给 Spring 容器托管。ApplicationEventPublisherAware 是由 Spring 提供的用于为 Service 注入 ApplicationEventPublisher 事件发布器的接口，使用这个接口，我们自己的 Service 就拥有了发布事件的能力。用户注册后，不再是显示调用其他的业务 Service，而是发布一个用户注册事件。

**例子1**

``` java
@Service
public class UserService  {
	
    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    public void register(String name) {
        System.out.println("用户：" + name + " 已注册！");
        applicationEventPublisher.publishEvent(new UserRegisterEvent(name));
    }
}
```

**例子2**

``` java
@Service
public class UserEventRegister {
    @Autowired
    private ApplicationEventPublisher publisher;
 
    public void register() throws Exception {
        UserDto user=new UserDto();
        user.setName("电脑");
        user.setSex("未知");
        publisher.publishEvent(user);
    }
}
```



### 事件订阅者

> 注意：事件订阅者的服务同样需要托管于 Spring 容器，ApplicationListener接口是由 Spring 提供的事件订阅者必须实现的接口，我们一般把该 Service 关心的事件类型作为泛型传入。处理事件，通过 event.getSource() 即可拿到事件的具体内容，在本例中便是用户的姓名。

**例子1**

``` java
@Service
public class EmailService implements ApplicationListener<UserRegisterEvent> {
    @Override
    public void onApplicationEvent(UserRegisterEvent userRegisterEvent) {
        System.out.println("邮件服务接到通知，给 " + userRegisterEvent.getSource() + " 发送邮件...");
    }
}
```

**例子2**

``` java
@Component
public class UserEventListener {
    @EventListener(condition = "#user.name!=null")
    public void handleEvent(UserDto user) throws Exception{
        System.out.println(user.getName());
        System.out.println(user.getSex());
    }
}

```



> EventObject，为JavaSE提供的事件类型基类，任何自定义的事件都继承自该类，例如上图中右侧灰色的各个事件。Spring中提供了该接口的子类ApplicationEvent。
>
> EventListener为JavaSE提供的事件监听者接口，任何自定义的事件监听者都实现了该接口，如上图左侧的各个事件监听者。Spring中提供了该接口的子类ApplicationListener接口。
>
> JavaSE中未提供事件发布者这一角色类，由各个应用程序自行实现事件发布者这一角色。Spring中提供了ApplicationEventPublisher接口作为事件发布者，并且ApplicationContext实现了这个接口，担当起了事件发布者这一角色。
>
> ApplicationContextAware提供了publishEvent()方法，实现Observer(观察者)设计模式的事件传播机，提供了针对Bean的事件传播功能。通过Application.publishEvent方法，我们可以将事件通知系统内所有的ApplicationListener。



Spring事件处理一般过程：

◆定义Event类，继承org.springframework.context.ApplicationEvent。
◆编写发布事件类Publisher，实现org.springframework.context.ApplicationContextAware接口。
◆覆盖方法setApplicationContext(ApplicationContext applicationContext)和发布方法publish(Object obj)。
◆定义时间监听类EventListener，实现ApplicationListener接口，实现方法onApplicationEvent(ApplicationEvent event)。

### 事件机制的SpringBoot的一般实现

**实现方法**

1. 自定义需要发布的事件类，需要继承ApplicationEvent类或PayloadApplicationEvent<T>(该类也仅仅是对ApplicationEvent的一层封装)
2. 使用@EventListener来监听事件
3. 使用ApplicationEventPublisher来发布自定义事件（@Autowired注入即可）

**注意事项**

1、监听器方法中一定要try-catch异常，否则会造成发布事件(有事物的)的方法进行回滚
2、可以使用@Order注解来控制多个监听器的执行顺序，@Order传入的值越小，执行顺序越高
3、对于需要进行事物监听或不想try-catch runtime异常，可以使用@TransactionalEventListener注解

**@TransactionalEventListener 监听器**

如果事件的发布不是在事物（@Transactional）范围内，则监听不到该事件，除非将fallbackExecution标志设置为true（@TransactionalEventListener(fallbackExecution = true)）;如果在事物中，可以选择在事物的哪个阶段来监听事件，默认在事物提交后监听。

>  如果需要异步则需要开启异步支持，在监听器方法加上@Async 注解即可。
