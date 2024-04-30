# JAVA8 debug 技巧

## 调试按钮说明

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499068947-f460c3fd-c3ef-4491-afe3-2b634262a20e.png)

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499070530-c0c628e6-7a56-444a-a191-0ba3dd980a24.png)

## Stream 调试

实例代码：

```java
// 1.Stream 调试
List&lt;Integer&gt; list1 = Lists.newArrayList(1, 5, 3, 2, 4);
List&lt;Integer&gt; list2 = list1.stream()
        .filter(o -&gt; o &gt; 2)
        .sorted()
        .collect(Collectors.toList());
System.out.println(list2);
```

在 stram() 处设置断点，debug 运行。

可以看到Debug视图中 Trace Current Steam Chain 图标会点亮。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499069843-06f36ac6-8d74-4221-ae28-635c665d27aa.png)

点击该图标弹出 Stream Trace 视图。可以看到 filter、sorted、collect 每一个方法执行之后的结果。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499069429-3bd7b09b-29c4-4d9f-9afd-07fcbc008a91.png)

ok，成功。



## Optional 调试

实例代码：

```java
// 2.Optional 调试
User user = new User()
        .setId(1)
        .setName(&#34;tom&#34;)
        .setArea(new Area().setProvince(&#34;beijing&#34;));

String province = Optional.ofNullable(user)
        .map(User::getArea)
        .map(Area::getProvince)
        .orElse(&#34;未知&#34;);
System.out.println(province);
```

在 Optaionl 处加断点，debug 运行，可以看到 Evaluate Express 图标。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499069962-409b0f79-b58e-45ca-8bf6-c44b21b7a783.png)

点击该图标弹出 Evaluate 视图，输入需要调试的代码，点击右下角 Evaluate 图标，可以看到执行结果。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499070454-e069bfaa-cb0a-4152-b548-0aefc97c62b5.png)

注：Optaionl 不像 Stream 那样，有专门的调试器，他需要借助 Evaluate Express 视图完成调试。



## 条件断点

实例代码：

```java
// 3.条件断点, 循环
List&lt;Integer&gt; integerList = Lists.newArrayList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
for (Integer x : integerList) {
    System.out.print(x);
}
```



有时候在循环中调试时，我们希望只关心我们需要关心的那个对象，但是一次循环记录有很多，每一个都跟一遍很是麻烦，那么条件断点就可以上场了。

在需要的地方加断点，在断点上右键弹出条件断点视图，输入条件表达式，点击 Done。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499070871-4c1848d9-49ed-4759-b3d4-912fee6a369c.png)

然后 debug 运行, 只有当 x 的值为 5 的时候才会停住，进入断点。



## 多线程调试

实例代码：

```java
// 4.多线程调试
new Thread(() -&gt; {
    System.out.println(&#34;线程1执行&#34;);
}, &#34;线程1&#34;).start();

new Thread(() -&gt; {
    System.out.println(&#34;线程2执行&#34;);
}, &#34;线程2&#34;).start();

System.out.println(&#34;主线程&#34;);
```



线程的执行时间是有系统分配，开发人员无法控制，但是通过Idea 专用的线程调试功能可以很好的选择要执行的线程代码，达到调试代码的目的。

先在线程内设置断点，在断点上右键选择 Thread 类型。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499071776-c8742633-1c45-4780-a71f-6fa9427e412e.png)

然后 debug 运行项目，在 Debug 的 Frames 视图可以方便的选择要执行的线程，选择后可继续一步步调试代码。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499071948-a700a80a-9491-46eb-bb4b-c148aaa5108a.png)





## 方法回退

实例代码：

```java
public static void method1() {
    System.out.println(&#34;method1&#34;);
    method2();
}

public static void method2() {
    System.out.println(&#34;method2&#34;);
    method3();
}

public static void method3() {
    System.out.println(&#34;method3&#34;);
}

public static void main(String[] args) {
    method1();
}
```



通过工具栏里的按钮可回退到上一步。

通过选中方法右键选择 Drop Frame 可回退到指定的方法。

如：阅读Spring IOC 启动流程源码时，方法调用都很深，想退回来再看看上一步做了啥，用方法回退很方便。

但是对于修改了共享数据的方法，如果回退再执行一次，可能产生异常数据。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499072085-26b843ff-3385-4f6e-aa54-88e3dcef190b.png)

## 强制返回



Debug 调试代码时，跟到一个无关痛痒的方法，有时候我们希望提前返回，可以使用下面的方法强制返回一个指定的值。

选择方法右键，选择Fore Return。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499072434-c1a5e750-1f2b-4193-ade0-e716953dda9c.png)

会弹出一个窗口，输入要返回的值即可。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499072285-8f20cdd7-a4cd-45ac-be2a-52035c06cea3.png)

## 热修改变量的值

有时候，在Debug 的过程中我们想调整一个参数的值又不想重启项目，可以使用Idea提供的实时修改变量值的功能。

首先在 Variables 视图选要修改的变量，右键选择Set Value。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499073255-7299d943-bd91-4986-82fa-fef5d29db917.png)

输入要修改的内容，回车可进行热修改。

![img](https://raw.gitmirror.com/telzhou618/images/main/img/1630499073224-eaf43192-cce7-48ed-a617-cfd7a7572f56.png)

---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/java8-debug/  

