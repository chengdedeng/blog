title: Squirrel使用
date: 2015-2-1 20:01:29
categories: 技术
tags: [Java,状态机]
----

>java状态机squirrel的初级用法

### Get Starting

**squirrel-foundation**既支持流式**API**又支持声明式创建状态机，并且还允许用户以一种简单方式定义操作方法。

* **StateMachine**接口需要以下4种泛型参数。
	* **T**代表实现的状态机类型。
	* **S**代表实现的状态类型。
	* **E**代表实现的事件类型。
	* **C**代表实现的外部上下文类型。

### State Machine Builder
	* State machine builder用来定义状态机。StateMachineBuilder能够通过StateMachineBuilderFactory来创建。
	* StateMachineBuilder由`*TransitionBuilder (InternalTransitionBuilder / LocalTransitionBuilder / ExternalTransitionBuilder)`（用于状态之间转换）和`EntryExitActionBuilder`（用于构建操作入口或出口）的组成。
	* Internal state(内部状态)会被隐式创建，在transition或者state action创建的时候。
	* 所有的状态机实例会被同一个状态机builder创建，该builder共享一份结构化的数据，从而优化内存的使用。
	* 状态机builder在生成状态机的时候使用lazy模式。当builder创建第一个状态机实例时，包含时间消耗的状态机定义才会被创建。但是状态机定义生成之后，接下来的状态机创建将会非常快。一句话，状态机builder应该尽量重用。

<!--more-->

为了创建一个状态机，首先得创建一个状态机builder。例如：

``` java
StateMachineBuilder<MyStateMachine, MyState, MyEvent, MyContext> builder=
StateMachineBuilderFactory.create(MyStateMachine.class, MyState.class, MyEvent.class, MyContext.class);
```

状态机builder应该包含machine(T)，state(S)，event(E)和context(C)几个参数。

-------

### Fluent API
状态机builder创建之后，我们就能使用流失API来定义状态机的state/transition/action。

``` java
builder.externalTransition().from(MyState.A).to(MyState.B).on(MyEvent.GoToB);
```
创建一个状态从状态A到B，并且MyEvent.GoToB事件触发的**external transition**。

``` java
builder.internalTransition(TransitionPriority.HIGH).within(MyState.A).on(MyEvent.WithinA).perform(myAction);
```
创建一个优先级为**TransitionPriority.HIGH**，内部状态为'A'，当事件为'WithinA'便执行'myAction'的状态机。`internal transition`意思是transition完成之后，没有状态退出和进入。优先级被用来覆盖继承来的状态机中的transition。

``` java
builder.externalTransition().from(MyState.C).to(MyState.D).on(MyEvent.GoToD).when(
    new Condition<MyContext>() {
        @Override
        public boolean isSatisfied(MyContext context) {
            return context!=null && context.getValue()>80;
        }

        @Override
        public String name() {
            return "MyCondition";
        }
}).callMethod("myInternalTransitionCall");
```
当external context满足条件限制时，就创建一个从状态'C'到状态'D'事件为MyEvent.GoToD的`conditional transition`，然后调用的action method为"myInternalTransitionCall"。

用户也可以采用如下的方式，使用[MVEL](http://mvel.codehaus.org/)(一个强大的描述性语言)来描述条件。

``` java
builder.externalTransition().from(MyState.C).to(MyState.D).on(MyEvent.GoToD).whenMvel(
    "MyCondition:::(context!=null && context.getValue()>80)").callMethod("myInternalTransitionCall");
```
**Note:**字符':::'用来分离条件名称和条件表达式。'context'是预先定义好的指向当前上下文的对象。

上面的事例代码列出了进入action的各种state。

------

### Method Call Action

用户可以在定义transition或者state进入/退出时，定义匿名action。然而action代码会散落在许多地方难以维护。
而且，别的用户不能重写这些action。所以squirrel-foundation支持状态机方法调用动作与状态机类本身一起定义。

``` java
StateMachineBuilder<...> builder = StateMachineBuilderFactory.create(
    MyStateMachine.class, MyState.class, MyEvent.class, MyContext.class);
builder.externalTransition().from(A).to(B).on(toB).callMethod("fromAToB");

// All transition action method stays with state machine class
public class MyStateMachine<...> extends AbstractStateMachine<...> {
    protected void fromAToB(MyState from, MyState to, MyEvent event, MyContext context) {
        // this method will be called during transition from "A" to "B" on event "toB"
        // the action method parameters types and order should match
        ...
    }
}
```
此外,squirrel-foundation也支持通过**Convention Over Configuration**(约定优于配置)来定义方法调用。基本上,这意味着,如果状态机中声明的方法满足命名公约和参数惯例,它将被添加到transition action列表,也会在特定的阶段调用。例如：

``` java
protected void transitFromAToBOnGoToB(MyState from, MyState to, MyEvent event, MyContext context)
```
方法名为`transitFrom[SourceStateName]To[TargetStateName]On[EventName]`，参数名为`[MyState, MyState, MyEvent, MyContext]`的方法会被添加到transition "A-(GoToB)->B"的action列表中。当状态机从'A'到'B'且触发的event为GoToB的时候，该方法会被调用。

```java
protected void transitFromAnyToBOnGoToB(MyState from, MyState to, MyEvent event, MyContext context)
```
方法`transitFromAnyTo[TargetStateName]On[EventName]`会在任何状态通过event 'GoToB'向状态'B'转化的时候调用。

```java
protected void exitA(MyState from, MyState to, MyEvent event, MyContext context)
```
方法**exit[StateName]**会在退出状态'A'的时候被调用。同理，**entry[StateName]**, **beforeExitAny/afterExitAny**和**beforeEntryAny/afterEntryAny**的意思也就不难理解了。

**还支持的命名模式：**

```java
transitFrom[fromStateName]To[toStateName]On[eventName]When[conditionName]  
transitFrom[fromStateName]To[toStateName]On[eventName]  
transitFromAnyTo[toStateName]On[eventName]  
transitFrom[fromStateName]ToAnyOn[eventName]  
transitFrom[fromStateName]To[toStateName]          
on[eventName]
```
上述这些方法规范还提供了类aop功能，squirrel在任何粒度提供了内置的灵活扩展能力。
如果需要获取更多信息，请参考测试用例`org.squirrelframework.foundation.fsm.ExtensionMethodCallTest`。
0.3.1版本后，还有另一种方法是通过流式API来定义这些类aop扩展方法(谢谢[vittali](https://github.com/vittali)的建议)，例如：

```java
// since 0.3.1
// the same effect as add method transitFromAnyToCOnToC in your state machine
builder.transit().fromAny().to("C").on("ToC").callMethod("fromAnyToC");
// the same effect as add method transitFromBToAnyOnToC in your state machine
builder.transit().from("B").toAny().on("ToC").callMethod("fromBToAny");
// the same effect as add method transitFromBToAny in your state machine
builder.transit().from("B").toAny().onAny().callMethod("fromBToAny");
```
或者使用注释，例如：

```java
// since 0.3.1
@Transitions({
     @Transit(from="B", to="E", on="*",   callMethod="fromBToEOnAny"),
     @Transit(from="*", to="E", on="ToE", callMethod="fromAnyToEOnToE")
})
```
**Note:**这些action方法会附加在已经存在且匹配上的transition上，但不创建新的transition。
从0.3.4，使用下面的API，可以一次定义多个transition。例如：

```java
// transitions(A->B@A2B=>a2b, A->C@A2C=>a2c, A->D@A2D) will be defined at once
builder.transitions().from(State._A).toAmong(State.B, State.C, State.D).
        onEach(Event.A2B, Event.A2C, Event.A2D).callMethod("a2b|a2c|_");

// transitions(A->_A@A2ANY=>DecisionMaker, _A->A@ANY2A) will be defined at once
builder.localTransitions().between(State.A).and(State._A).
        onMutual(Event.A2ANY, Event.ANY2A).
        perform( Lists.newArrayList(new DecisionMaker("SomeLocalState"), null) );
```
更多的信息可以查看`org.squirrelframework.foundation.fsm.samples.DecisionStateSampleTest`。

-----

### Declarative Annotation
该方式提供了一种声明式的方式来定义和扩展状态机。例子如下：

```java
@States({
    @State(name="A", entryCallMethod="entryStateA", exitCallMethod="exitStateA"),
    @State(name="B", entryCallMethod="entryStateB", exitCallMethod="exitStateB")
})
@Transitions({
    @Transit(from="A", to="B", on="GoToB", callMethod="stateAToStateBOnGotoB"),
    @Transit(from="A", to="A", on="WithinA", callMethod="stateAToStateAOnWithinA", type=TransitionType.INTERNAL)
})
interface MyStateMachine extends StateMachine<MyStateMachine, MyState, MyEvent, MyContext> {
    void entryStateA(MyState from, MyState to, MyEvent event, MyContext context);
    void stateAToStateBOnGotoB(MyState from, MyState to, MyEvent event, MyContext context)
    void stateAToStateAOnWithinA(MyState from, MyState to, MyEvent event, MyContext context)
    void exitStateA(MyState from, MyState to, MyEvent event, MyContext context);
    ...
}
```
注解既可以定义在状态机的实现类上也可以定义在状态机需要实现的任何接口上面。注解也可以混合着流式API使用，这意味着流式API定义的状态机也可以使用注解。（有一件事情需要注意，接口中定义的方法必须要是public的，这意味着对于方法的调用者来说，需要调用的方法在实现类中必须是public的。）

------

### Converters
为了在注解@State和@Transit中声明状态，用户需要为state（S）和event（E）类型实现相应的转换器。该转换器必须实现Converter<T>接口，该转换器可以把state/event和String进行互相转换。

```java
public interface Converter<T> extends SquirrelComponent {
    /**
    * Convert object to string.
    * @param obj converted object
    * @return string description of object
    */
    String convertToString(T obj);

    /**
    * Convert string to object.
    * @param name name of the object
    * @return converted object
    */
    T convertFromString(String name);
}
```
然后再将这些转换器注册到ConverterProvider中，例如：

```java
ConverterProvider.INSTANCE.register(MyEvent.class, new MyEventConverter());
ConverterProvider.INSTANCE.register(MyState.class, new MyStateConverter());
```
**注意：**如果你仅使用流式API来定义状态机，那么不需要实现相应的转换器。并且如果Event和State是String和Enumeration类型，在大多数情况下你也不需要实现和注册转换器。

----

### New State Machine Instance
用户定义完状态机的行为之后，就需要通过builder来创建一个状态机实例。注意，一旦通过builder创建好状态机，该builder就不能再用于定义任何包含新元素的状态机。

```java
T newStateMachine(S initialStateId, Object... extraParams);
```
为了通过状态机builder创建状态机实例，你需要传递以下的参数。

	i. initialStateId:开始时，状态机的初始状态。
	ii. extraParams:创建状态机时需要的额外参数。如果不需要额外的参数，设置为"new Object[0]"。
		a.如果在创建状态机实例时传递了额外的参数，请确保在创建状态机builder时定义了StateMachineBuilderFactory

假如不需要传递额外参数，用户可以简单的调用**T newStateMachine(S initialStateId)**来创建状态机实例。

通过状态机builder创建一个新的状态机。（该例子，不需要传递额外的参数。）

```java
MyStateMachine stateMachine = builder.newStateMachine(MyState.Initial);
```

------

### Trigger Transitions

状态机创建之后，用户可以发送event以及context，在状态机内部触发transition。例如：

```java
stateMachine.fire(MyEvent.Prepare, new MyContext("Testing"));   
```
### Untyped State Machine

UntypedStateMachine的目的是为了简化状态机的使用，和避免由于太多的泛型（如：StateMachine<T,S,E,C>）造成的代码难以阅读的情况，并且仍然能够保证transition重要部分的类型安全。

```java
enum TestEvent {
    toA, toB, toC, toD
}

@Transitions({
    @Transit(from="A", to="B", on="toB", callMethod="fromAToB"),
    @Transit(from="B", to="C", on="toC"),
    @Transit(from="C", to="D", on="toD")
})
@StateMachineParameters(stateType=String.class, eventType=TestEvent.class, contextType=Integer.class)
class UntypedStateMachineSample extends AbstractUntypedStateMachine {
    // No need to specify constructor anymore since 0.2.9  
    // protected UntypedStateMachineSample(ImmutableUntypedState initialState,
    //  Map<Object, ImmutableUntypedState> states) {
    //    super(initialState, states);
    // }

    protected void fromAToB(String from, String to, TestEvent event, Integer context) {
        // transition action still type safe ...
    }

    protected void transitFromDToAOntoA(String from, String to, TestEvent event, Integer context) {
        // transition action still type safe ...
    }
}

UntypedStateMachineBuilder builder = StateMachineBuilderFactory.create(
    UntypedStateMachineSample.class);
// state machine builder not type safe anymore
builder.externalTransition().from("D").to("A").on(TestEvent.toA);
UntypedStateMachine fsm = builder.newStateMachine("A");
```
为了创建一个UntypedStateMachine，首先需要通过StateMachineBuilderFactory创建一个UntypedStateMachineBuilder。StateMachineBuilderFactory仅需要一个参数，该参数就是需要创建UntypedStateMachineBuilder的类的名称。注解@StateMachineParameters用来声明状态机泛型参数类型。AbstractUntypedStateMachine是任何无状态的状态机的基类。

### Context Insensitive State Machine

有时状态机根本不关心上下文，意思是transition只取决于事件。这种情况下用户可以使用上下文不敏感的状态机来简化方法调用的参数。

声明上下文不敏感的状态机非常简单。用户只需要在状态机实现类上添加注解@ContextInsensitive。之后，在transition方法的参数中可以忽略context参数。例如：

```java
@ContextInsensitive
public class ATMStateMachine extends AbstractStateMachine<ATMStateMachine, ATMState, String, Void> {
    // no need to add context parameter here anymore
    public void transitFromIdleToLoadingOnConnected(ATMState from, ATMState to, String event) {
        ...
    }
    public void entryLoading(ATMState from, ATMState to, String event) {
        ...
    }
}
```

----

### Transition Exception Handling

当状态转换过程中出现异常，已执行的action列表将失效并且状态机会进入error状态，意思就是状态机实例不会再处理任何event。假如用户继续向状态机发送event，便会抛出**IllegalStateException**异常。所有状态转换过程中发生的异常，包括action执行和外部listener调用，会被包装成TransitionException（未检查异常）。目前，默认的异常处理策略非常简单并且粗暴的连续抛出异常，可以参阅`AbstractStateMachine.afterTransitionCausedException`方法。

```java
protected void afterTransitionCausedException(...) { throw e; }
```
假如状态机可以从该异常恢复，用户可以扩展`afterTransitionCausedException`方法，在该方法中添加相应的恢复逻辑。**注意**，在最后别忘记将状态设置为正常。

```java
@Override
protected void afterTransitionCausedException(Object fromState, Object toState, Object event, Object context) {
    Throwable targeException = getLastException().getTargetException();
    // recover from IllegalArgumentException thrown out from state 'A' to 'B' caused by event 'ToB'
    if(targeException instanceof IllegalArgumentException &&
            fromState.equals("A") && toState.equals("B") && event.equals("ToB")) {
        // do some error clean up job here
        // ...
        // after recovered from this exception, reset the state machine status back to normal
        setStatus(StateMachineStatus.IDLE);
    } else if(...) {
        // recover from other exception ...
    } else {
        super.afterTransitionCausedException(fromState, toState, event, context);
    }
}
```

------

### Define Hierarchical State

层级状态可以包含嵌套的状态。子状态自己可以包含嵌套的子状态和嵌套任意深度的逻辑。当一个嵌套状态处于活跃，有且仅有一个子状态处于活跃。分层状态可以通过API或者注解来定义。

```java
void defineSequentialStatesOn(S parentStateId, S... childStateIds);
```
`builder.defineSequentialStatesOn(State.A, State.BinA, StateCinA)`定义了两个子状态“BinA”和“CinA”从属于父母状态“A”，第一个定义的子状态"A"也就是初始状态。同一个层级状态也可以通过注解来定义，例如：

```java
@States({
    @State(name="A", entryMethodCall="entryA", exitMethodCall="exitA"),
    @State(parent="A", name="BinA", entryMethodCall="entryBinA", exitMethodCall="exitBinA", initialState=true),
    @State(parent="A", name="CinA", entryMethodCall="entryCinA", exitMethodCall="exitCinA")
})
```

### Define Parallel State

并行状态是将孩子状态封装到一个集合中，当父元素激活时，并行状态便激活。并行状态既可以通过API定义又可以通过注解定义。例如：
![Parallel_State](/images/Parallel_State.png)

```java
// defines two region states "RegionState1" and "RegionState2" under parent parallel state "Root"
builder.defineParallelStatesOn(MyState.Root, MyState.RegionState1, MyState.RegionState2);

builder.defineSequentialStatesOn(MyState.RegionState1, MyState.State11, MyState.State12);
builder.externalTransition().from(MyState.State11).to(MyState.State12).on(MyEvent.Event1);

builder.defineSequentialStatesOn(MyState.RegionState2, MyState.State21, MyState.State22);
builder.externalTransition().from(MyState.State21).to(MyState.State22).on(MyEvent.Event2);
```
或

```java
@States({
    @State(name="Root", entryCallMethod="enterRoot", exitCallMethod="exitRoot", compositeType=StateCompositeType.PARALLEL),
    @State(parent="Root", name="RegionState1", entryCallMethod="enterRegionState1", exitCallMethod="exitRegionState1"),
    @State(parent="Root", name="RegionState2", entryCallMethod="enterRegionState2", exitCallMethod="exitRegionState2")
})
```
从并行状态获取当前子状态

```java
stateMachine.getSubStatesOn(MyState.Root); // return list of current sub states of parallel state
```
当所有的并行状态到达最终状态，一个**Finish**上下文的event会被发送。

### Define Context Event

状态机中的上下文事件都有预定义的上下文。squirrel-foundation为不同使用实例定义了3种上下文事件。

**Start/Terminate Event**：当状态机启动/终止时，会使用到定义好的start/terminate事件。因此用户可以辨认action触发时的调用，例如。当状态机启动和进入起始状态，用户可以知道被start事件激活的action调用。

**Finish Event**：当所有的并行状态抵达最终状态，finish event会被自动发射。用户可以在finish event上定义如下如下的转换。

定义上下文事件，也有两种方法，注解获取builder API。

```java
@ContextEvent(finishEvent="Finish")
static class ParallelStateMachine extends AbstractStateMachine<...> {
}
```
或

```java
StateMachineBuilder<...> builder = StateMachineBuilderFactory.create(...);
...
builder.defineFinishEvent(HEvent.Start);
builder.defineTerminateEvent(HEvent.Terminate);
builder.defineStartEvent(HEvent.Finish);
```
