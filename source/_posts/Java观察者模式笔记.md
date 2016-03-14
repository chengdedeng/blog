title: Java观察者模式笔记
date: 2014-9-2 17:37:30
categories: 技术
tags: [Java,设计]
------

>Java观察者模式的实现与对比

观察者模式作为一个耳熟能详的设计概念，早已被绝大多数的程序员所熟悉。虽然`Observer`和`Observable`从**JDK1.0**起就已经存在，但是到现在它们还是那样的难用，至少和`Guava`的`EventBus`比较起来。传统的发布订阅模式，主要是为了解决进程内事件的分发，从而去掉了显示的注册方式，从而是组件之间可以更好的解耦。现在的发布订阅随着消息中间件的流行，早已实现了跨进程通信。本文主要是对比**JDK**和**Guava**观察者模式的实现区别，如果是玩**Android**的朋友，还可以看看[square/otto](https://github.com/square/otto)(专门为Android平台进行了优化的Guava EventBus库)、[greenrobot/EventBus](https://github.com/greenrobot/EventBus)。

<!--more-->

### JDK观察者模式实现

``` java
public class Student extends Observable {
    public String name;
    private String course;
    public final static String ENGLISH = "english";
    public final static String HISTORY = "history";
    public final static String MATH = "math";

    public Student(String name) {
        this.name = name;
    }

    public String getState() {
        return course;
    }

    public void changeState(String course) {
        if (this.course != course) {
            this.course = course;
            this.setChanged();
            if (ENGLISH == course) {
                this.notifyObservers("英语课");
            } else if (HISTORY == course) {
                this.notifyObservers("历史课");
            } else if (MATH == course) {
                this.notifyObservers("数学课");
            }
        } else {
            this.notifyObservers("同上一节课");
        }
    }
}
```

``` java
public class Teacher implements Observer {
    private String name;

    public Teacher(String name) {
        super();
        this.name = name;
    }

    public void update(Observable o, Object arg) {
        Student student = (Student) o;
        //获取被观察对象当前的状态
        System.out.println(name + "被通知：" + student.name + "正在上" + student.getState() + "课");
    }
}
```

``` java
public class ObserverDemo {
    public static void main(String[] args) {
        //被观察者
        Student student = new Student("杨果");
        //观察员：李老师
        Teacher teacher1 = new Teacher("李老师");
        //观察员：王老师
        Teacher teacher2 = new Teacher("王老师");
        //观察员：陈老师
        Teacher teacher3 = new Teacher("陈老师");
        //向被观察对象注册观察员
        //为学生注册观察员：李老师，王老师，陈老师
        student.addObserver(teacher1);
        student.addObserver(teacher2);
        student.addObserver(teacher3);
        //更改被观察对象的状态
        student.changeState(Student.HISTORY);
        student.changeState(Student.MATH);
        student.changeState(Student.ENGLISH);
    }
}
```
--------

### Guava观察者实现

``` java
public class EventBusTest {
    public class EventListener1 {
        @Subscribe
        public void subscribe(String message) {
            System.out.println("Event1:" + message);
        }
    }

    public class EventListener2 {
        @Subscribe
        public void subscribe(String message) {
            System.out.println("Event2:" + message);
        }
    }

    public class EventListener3 {
        @Subscribe
        public void subscribe(String message) {
            System.out.println("Event3:" + message);
        }
    }

    public class EventListener4 {
        @Subscribe
        public void subscribe(String message) {
            System.out.println("Event4:" + message);
        }
    }

    /**
     * 注意继承问题
     */
    public class MultipleListener {
        public Integer lastInteger;
        public Long lastLong;
        public Number lastNumber;

        @Subscribe
        public void listenInteger(Integer event) {
            System.out.println("Integer:" + event);
            lastInteger = event;
        }

        @Subscribe
        public void listenLong(Long event) {
            System.out.println("Long:" + event);
            lastLong = event;
        }

        @Subscribe
        public void listenNumber(Number event) {
            System.out.println("Number:" + event);
            lastNumber = event;
        }

        public Integer getLastInteger() {
            return lastInteger;
        }

        public Long getLastLong() {
            return lastLong;
        }
    }

    public class DeadEventListener {
        @Subscribe
        public void listen(DeadEvent event) {
            System.out.println(event.getEvent());
            System.out.println(event.getSource());
        }
    }

    /**
     * 测试同步事件总线
     */
    @Test
    public void testSyncEventBus() {
        EventBus eventBus = new EventBus();
        eventBus.register(new EventListener1());//注册事件
        eventBus.register(new EventListener2());//注册事件
        eventBus.register(new EventListener3());//注册事件
        eventBus.register(new EventListener4());//注册事件
        eventBus.post("hello word");// 触发事件处理

        System.out.println("Game over");
    }

    /**
     * 测试异步事件总线
     */
    @Test
    public void testAysncEventBus() {
        AsyncEventBus eventBus = new AsyncEventBus(Executors.newFixedThreadPool(3));
        eventBus.register(new EventListener1());
        eventBus.register(new EventListener2());
        eventBus.register(new EventListener3());
        eventBus.register(new EventListener4());
        eventBus.post("hello word");

        System.out.println("Game over");
    }

    /**
     * 测试多事件监听器
     */
    @Test
    public void testMultipleEventBus() {
        EventBus eventBus = new EventBus();
        MultipleListener multiListener = new MultipleListener();

        eventBus.register(multiListener);

        eventBus.post(new Integer(100));
        eventBus.post(new Long(800));

        Assert.assertEquals(multiListener.getLastInteger(), new Integer(100));
        Assert.assertEquals(multiListener.getLastLong(), new Long(800L));
    }

    /**
     * 测试Dead Event
     */
    @Test
    public void testDeadEventBus() {
        EventBus eventBus = new EventBus();

        DeadEventListener deadEventListener = new DeadEventListener();
        eventBus.register(deadEventListener);

        eventBus.post("hello word");
    }
}

```
