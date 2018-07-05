# Android无埋点用户行为统计

## 前言

我们知道，要统计app上用户的行为，是需要根据需求在功能点对应的方法中加上一些代码的。现在我们是通过调用Ums框架中的一句代码来实现的，不调用显然就不能进行用户行为统计。**所以，无论是现有埋点方法，还是所谓的无埋点方案，都是要调用一句统计代码才可以进行用户行为统计的。**所以我们要解决的问题实际上是如何写一个插件实现自动插入代码。

## 一、Android打包流程

下图是android的打包流程，想要插入代码，可以在.java文件->.class文件和.class文件->.dex文件这两个过程中实现。

![img](https://img-blog.csdn.net/20161213230703575?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhhb2RlY2FuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



### 1.1 .java->.class期间生成代码

从图中可以看到，生成class文件是需要java编译期参与的，所以在这个过程中，我们想生成代码，就需要依靠java编译器（或一个可以编译出符合java class 语法的的编译器）。

### 1.2 .class->.dex期间生成代码

图中“dex”节点，表示将class文件打包到dex文件的过程，其输入包括

（1）项目java源文件经过javac后生成的class文件

（2）第三方依赖的class文件

> class文件是二进制格式的（class文件是一种紧凑的8位字节的二进制流文件， 各个数据项按顺序紧密的从前向后排列， 相邻的项［包括字节码指令］之间没有间隙），我们就是通过对这个二进制数据进行扫描，按照一定规则过滤然后修改字节码的。

## 二、实现自动插入代码的几种方案

### 2.1 AnnotationProcess

最先想到的是基于Java Compiler，通过注解的方式，用**AnnotationProcess**（注解处理器）生成代码。

#### 2.1.1 主要原理

在编译期间，以注解中的参数作为输入，生成文件.java文件作为输出。**但是，这些生成的Java代码是在生成的.java文件中，所以不能修改已经存在的Java类，例如不能向已有的类中添加方法。**也就是说，用这种方式依然需要**手动在代码中调用编译期间生成的类的方法**。而无埋点恰恰需要的是在已有的方法中添加代码。所以用这种方式做无埋点并不可行。

#### 2.1.2 技术用途

虽然AnnotationProcess无法满足我们的需求，但是为了加深理解，这里举了一个例子来说明该技术的用途。

这里介绍一个Android中比较有名依赖注入框架开源框架ButterKnife。这个框架的主要作用是框架自动生成findviewById和setOnClickListener等方法，无需我们关系view对象的创建过程。

为了阐述原理，我对ButterKnife的源码进行了简化提取，写了一个Demo。

这是框架的使用方法：

```java
public class BaseActivity extends Activity {
    @ViewInjector(R.id.txt_test)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //在BaseActivity中调用一句代码
        ButterKnife.inject(this);
    }

}
```

这是注解的源码，可以看出这是一个编译期注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.CLASS)
public @interface ViewInjector {
    int value();
} 
```

那么编译器是如何在编译期间找到注解的呢，这时候就要用到AnnotationProcess了，这个类的主要作用就是扫描注解，然后根据不同的注解或者注解中不同的参数，进行对应的代码生成的操作。

```java
@AutoService(Processor.class)
@SupportedAnnotationTypes("com.geekholt.annotation.ViewInjector")
@SupportedSourceVersion(SourceVersion.RELEASE_7)
public class ViewInjectProcessor extends AbstractProcessor {

    //注册处理器集合
    // 1.会有多种注解需要处理（视图、事件）;
    // 2.视图注入可以有不同的处理方式
    List<AnnotationHandler> mHandlers = new ArrayList<>();
    Map<String, List<VariableElement>> mMap = new HashMap<>();
    AdapterWriter mWriter;

    //初始化：
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        //1.初始化注解处理器
        registerHandler(new ViewInjectHandler());
        //2.初始化辅助类生成器
        mWriter = new DefaultAdapterWriter(processingEnv);
    }

    protected void registerHandler(AnnotationHandler handler) {
        mHandlers.add(handler);
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {

        //遍历处理器集合
        for (AnnotationHandler handler : mHandlers) {
            //关联属性，processingEnv是父类中的属性
            handler.attachProcessingEnvironment(processingEnv);
            //处理注解（得到哪些类有哪些属性需要注入的列表)
            mMap.putAll(handler.handleAnnotation(roundEnvironment));
        }

        //生成辅助类
        mWriter.generate(mMap);

        return true;
    }
}
```

编译期生成的class文件是这样的，生成了一个BaseActivity的内部类。（内部类的class文件和主类是分开的两个文件）

```java
//BaseActivity的内部类
public class BaseActivity$InjectAdapter implements InjectAdapter<BaseActivity> {
    public BaseActivity$InjectAdapter() {
    }

  	//实际创建view的对象的方法
    public void injects(BaseActivity target) {
        target.textView = (TextView)ViewFinder.findViewById(target, 2131165307);
    }
}
```

也就是说，我们在BaseActivity中调用的 ButterKnife.inject(this) 这句代码会去调用上面这个在编译期生成的代码。那么如何调用还没有生成的代码呢，我们可以在运行时通过反射来创建内部类对象，并调用它所实现的接口的方法。

```java
public class ButterKnife {
    private static final String SUFFIX = "$InjectAdapter";
	//用来存储编译期生成的类的容器
    static Map<Class<?>, InjectAdapter<?>> mInjectCache = new HashMap<Class<?>, InjectAdapter<?>>();

    public static void inject(Activity target) {
        //InjectAdapter就是生成类要实现的接口
        InjectAdapter injectAdapter = getViewAdapter(target.getClass());
        injectAdapter.injects(target);
    }

    private static <T> InjectAdapter getViewAdapter(Class<? extends Activity> clazz) {
        InjectAdapter<T> adapter = (InjectAdapter<T>) mInjectCache.get(clazz);
        if (adapter != null) {
            //如果缓存容器中存在class对应的对象，直接返回
            return adapter;
        }
        String adapterClassName = clazz.getName() + SUFFIX;
        try {
            Class<?> adapterClazz = Class.forName(adapterClassName);
            //通过反射创建内部类对象
            adapter = (InjectAdapter<T>) adapterClazz.newInstance();
            mInjectCache.put(clazz, adapter);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

        return adapter;
    }
}
```

综合上述，ButterKnife实现的是**依赖注入**，即创建对象的过程。依赖注入是**控制反转**（Inversion of Control）的一部分（控制反转包括依赖注入和依赖查找），这是一种全新的设计模式，但是由于理论和时间成熟相对较晚，并没有包含在GoF中。目前在Android中运用还是较少，但这在J2EE的Spring框架中是非常重要的一部分，控制反转就是将创建对象的权利交给框架，这样可以的降低模块之间的耦合度。

我们现在要实现的是在方法中插入代码，那么就需要引入AOP这个概念。

### 2.2 AOP介绍

AOP（Aspect Oriented Programming）是面向切面编程的简称。这是一种完全不同OOP的设计思想。OOP（面向对象编程）针对业务处理过程的**实体及其属性和行为进行抽象封装**，以获得更加清晰高效的逻辑单元划分。 而AOP则是针对业务处理过程中的切面进行提取，它所面对的是**处理过程中的某个步骤**，以获得逻辑过程中各部分之间低耦合性的隔离效果。![img](https://upload-images.jianshu.io/upload_images/1417134-d0db3ad64dc58ab0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

上图是一个APP模块结构示例，按照OOP的思想划分为“视图交互”，“业务逻辑”，“网络”等三个模块，而现在假设想要对所有模块的每个方法耗时（性能监控模块）进行统计。这个性能监控模块的功能就是需要横跨并嵌入众多模块里的，这就是典型的AOP的应用场景。

#### 2.2.1 用动态代理模式实现AOP

JDK动态代理模式关键代码：

```java
public class JDKDynamicProxy implements InvocationHandler {         
        private Object target;    
       
        public JDKDynamicProxy(Object target) {    
            this.target = target;    
        }    
       
        @SuppressWarnings("unchecked")    
        public <T> T getProxy() {    
            return (T) Proxy.newProxyInstance(    
                target.getClass().getClassLoader(),    
                target.getClass().getInterfaces(),    
                this   
            );    
        }    
       
        @Override   
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {    
            before();    
            Object result = method.invoke(target, args);    
            after();    
            return result;    
        }    
       
        private void before() {    
            System.out.println("Before");    
        }    
       
        private void after() {    
            System.out.println("After");    
        }    
}
```

客户端调用

```java
public class ProxyTest {
    public static void main(String[] args) {
        
        //创建一个被代理对象，Person是一个接口
        Person zhangsan = new Student("张三");
        
        //创建一个与代理对象相关联的InvocationHandler
        JDKDynamicProxy proxy = new JDKDynamicProxy(zhangsan);
        
        //创建一个代理对象stuProxy来代理zhangsan，代理对象的每个执行方法都会替换执行Invocation中的invoke方法
        Person stuProxy = (Person) proxy.getProxy()；

       //代理执行上交班费的方法
        stuProxy.giveMoney();
    }
}
```

JDK的动态代理是靠多态和反射来实现的，它生成的代理类需要实现你传入的接口，并通过反射来得到接口的方法对象，并将此方法对象传参给增强类的invoke方法去执行，从而实现了代理功能。

动态代理模式存在两个问题：

1.接口是jdk动态代理的核心实现方式，没有接口就无法通过反射找到方法

2.反射只运行时方法，性能较差。

针对第一点，在Android中比较好的一点是，几乎所有我们要监听的用户行为都实现了OnCLickListener接口，但是问题在于，onClick是一个回调方法，我们并不能在源码中调用OnClickListener.onClick( )方法时对OnClickListener进行代理。所以动态代理模式并不符合我们的需求。

#### 2.2.2 用AspectJ实现AOP

AspectJ是一个代码生成框架，它扩展了Java语言，定义了AOP语法，所以它有一个**专门的编译器用来生成遵守Java字节编码规范的Class文件**。

AspectJ的使用方法是给需要进行埋点的方法添加一个注解，然后AspectJ会在编译期扫描包含注解的类，并生成class文件。所以说AspectJ扩展了java语言，能够将编译时生成的代码插入到现有的类中。

```java
    @BehaviorTrace("语音通话")
    public void audio(View btn) {
        Log.i("geekholt", "语音通话中...");
        SystemClock.sleep(20);
    }
```

AspectJ的编译器会在编译期间找到带有@Aspect注解的类，并执行。

```java
@Aspect
public class BehaviorAspect {
    private static final String POINTCUT_METHOD =
            "execution(@com.geekholt.aspectj.annotation.BehaviorTrace * *(..))";

    //任何一个包下面的任何一个类下面的任何一个带有BehaviorTrace的方法，构成了这个切面
    @Pointcut(POINTCUT_METHOD)
    public void annoHaviorTrace() {
        Log.i("tag", "annoHaviorTrace");
    }

    //拦截方法
    @Around("annoHaviorTrace()")
    public Object weaveJoinPoint(ProceedingJoinPoint joinPoint) throws Throwable {
        Log.i("tag", "weaveJoinPoint");
        //拿到方法的签名
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        //类名
        String className = methodSignature.getDeclaringType().getSimpleName();
        //方法名
        String methodName = methodSignature.getName();
        //功能名
        BehaviorTrace behaviorTrace = methodSignature.getMethod().getAnnotation(BehaviorTrace.class);
        String fun = behaviorTrace.value();

        //方法执行前
        long begin = System.currentTimeMillis();

        //执行拦截方法
        Object result = joinPoint.proceed();

        //方法执行后
        long duration = System.currentTimeMillis() - begin;
        Log.i("tag", String.format("功能：%s，%s的%s方法执行，耗时：%d ms", fun, className, methodName, duration));
        return result;
    }
}

```

这个方法的优点是：

（1）代码的侵入性较低。

（2）相对于代码埋点更佳优雅。

（3）可以通过注解属性传参的方式确定该埋点所对应的业务。

但是该方法依然没有实现无埋点，只是将代码埋点转变成了注解埋点。

#### 2.2.3 用ASM实现AOP

那么我们能不能不用注解，找出需要统计的这些方法的共同特点，将这个共同特点作为一种标注呢？显然，如果我们要统计的用户点击行为，基本都要实现onClickListener接口的onClick方法。如果我们要统计页面停留时长，我们就要实现Activity中onResume或Page中的onForeground方法。所以我们要解决的，就是如何在编译期对实现了指定接口的指定方法进行代码插入。

既然.java文件->.class文件这个过程不能满足我们在方法中插入代码的需求。那么我们可以尝试在class文件->dex文件处进行拦截，直接修改字节码文件。

于是就有了如下的思路：

（1）Gradle Transform API 拦截class文件
Google官方在Android Gradle的1.5.0 版本以后提供了 Transfrom API, 允许在打包dex 文件之前操作class 文件，所以可以用Transform API进行class文件遍历拿到所有方法。

（2）Gradle Plugin
通过Transform提供的api可以遍历所有文件，但是要实现Transform的遍历操作，需要实现Gradle的Plugin类，所以这只是一个编译期的脚本，并不会添加依赖到主项目中。

（3）字节码编写
拿到class文件和所有相关方法后，就需要进行class文件进行修改，class文件是字节码格式的，操作起来难度很大，所以需要一个字节码操作库来减轻难度，这个库就是ASM。ASM提供API可以在需要改写的方法的前面或后面插入代码，也可以同时插入。

（4）对需要改写的方法添加两个筛选条件，一是方法的名字相同，二是方法所在类或接口相同。例如：需要对用户的所有点击事件做监听，那么就需要找到实现了"android/view/View$OnclickListener"的类中的onClick方法

```java
  public static MethodVisitor getMethodVisitor(String[] interfaces, String className,
                                          MethodVisitor methodVisitor,
                                        int access, String name, String desc) {
        MethodVisitor adapter = null；
          //当方法名为onClick，接口名为android/view/View$OnClickListener才执行代码插入
        if (name.equals("onClick") && isMatchingInterfaces(interfaces, 'android/view/View$OnClickListener')) {
            adapter = new AutoMethodVisitor(methodVisitor, access, name, desc) {
                @Override
                protected void onMethodEnter() {
                    super.onMethodEnter()；
                      //在方法执行前插入代码
                    methodVisitor.visitVarInsn(Opcodes.ALOAD, 1)；
                    //调用AutoHelper.onClick(view)方法
                    //AutoHelper是我们在Ums框架中事先写好的统计代码
                    methodVisitor.visitMethodInsn(Opcodes.INVOKESTATIC, "com/xishuang/plugintest/AutoHelper", "onClick", "(Landroid/view/View;)V", false)；
                }
                @Override
                protected void onMethodEnter() {
                    super.onMethodEnter()；
                 //在方法执行后插入代码
                      ....
                }
            }
        } 
        return adapter；
    }

```

> ASM API 实际上是封装了JVM指令。用ASM框架能够减轻编写JVM指令的难度，但是对于不熟悉JVM指令的人，门槛依然是很高的。这里有一个办法，我们能够先在一个类中编写你想自动生成的代码，然后反编译（到命令javap -c  TestActivity.class）这个类class文件，查看生成这个类的JVM指令，这样就可以找到你想生成的代码的JVM指令，就可以仿造着写了。

所以，现在我们已经可以真正意义上的实现在编译期在任意类的任意方法中插入代码了。

不过插入的代码也是有一定限制的，比如**方法中只能调用当前类的引用和当前方法的入参的引用**（比如onClick方法中的view），还能调用一些**全局的静态方法**，就和正常我们能在该方法中写的代码的规则是一样的。

## 三、如何确定插入的代码对应的业务

我们现在已经解决了如何自动插入代码的问题，但是，我们如何才能知道当前onClick方法中插入的代码统计的具体是什么业务呢？

我们知道，在编译期间，我们只能通过SDK的方法拿到数据，比如当前页面的类名，当前点击的view的id等等，不可能拿到我们需要的业务名称。这样看起来，无埋点似乎是一个不可能实现的需求。

事实也确实如此，我们理想中的“无埋点”依靠现有的技术根本无法实现。那么所谓的无埋点，其实在进行到这一步的时候，已经受到了限制。

那么我们能通过view的某种属性（唯一标识），来确定这个view对应的业务么？

### 3.1 确定View的唯一标识

#### 3.1.1 view.getId()

首先最容易想到的，view.getId()即可获得一个int型的id用于区分view,但是这个ID因为以下两个原因却并不能满足我们的需要。

（1）有相当一部分view是NO_ID，比如在布局文件中未指定id,或者直接在代码里面new出来view，view.getId()返回的全部都是NO_ID 。

（2）这个ID是不稳定的，由于这个ID其实就是每次编译产生的R文件中的int常量，因此同一个按钮，两个版本编译出来的ID很可能时不一样的。

#### 3.1.2 viewPath

每个Window(ActivityWindow/DialogWindow/PopupWindow等)上面都生长着一棵ViewTree.而屏幕中看到的各种控件(ImageView/Button等)都是这棵ViewTree上的节点。同时每个节点都包含**纵向的深度**和**横向的index**。

这是构建viewPath的方法

```java
 public static String getPath(Context context, View childView) {
        StringBuilder builder = new StringBuilder();
        String viewType = childView.getClass().getSimpleName();
        View parentView = childView;
        int index;
        // 遍历view获取父view来进行拼接
        do {
            int id = childView.getId();
            index = ((ViewGroup) childView.getParent()).indexOfChild(childView);
            // 根据从属于不同的类进行index判断
            if (childView.getParent() instanceof RecyclerView) {
                index = ((RecyclerView) childView.getParent()).getChildAdapterPosition(childView);
            } else if (childView.getParent() instanceof AdapterView) {
                index = ((AdapterView) childView.getParent()).getPositionForView(childView);
            } else if (childView.getParent() instanceof ViewPager) {
                index = ((ViewPager) childView.getParent()).getCurrentItem();
            }

            builder.insert(0, getResourceId(context, childView.getId()));
            builder.insert(0, "]");
            builder.insert(0, index);
            builder.insert(0, "[");
            builder.insert(0, viewType);

            parentView = (ViewGroup) parentView.getParent();
            viewType = parentView.getClass().getSimpleName();
            childView = parentView;
            builder.insert(0, "/");
        } while (parentView.getParent() instanceof View);

        builder.insert(0, getResourceId(context, childView.getId()));
        builder.insert(0, viewType);
        return builder.toString();
    }
```

viewPath看似唯一，但是可能还会存在一些问题，所以只能是基于viewPath，然后给viewPath加一些tag，比如view的资源名称，当前的frameId等等。目前没有发现viewPath重复的情况，但是为了更加严谨，之后这个唯一标识还可以进行更多的优化。

### 3.2 view的唯一标识如何对应业务

最初的想法是，客户端点击一个按钮，然后在日志打印一个view唯一标识。再手动将这个唯一标识记录到一个配置文件中。然后针对每个需要埋点的view做这个操作，这样就生成了一整份配置表，然后将配置表放到服务端。客户端实际上做了一个全埋点，即每次点击都把唯一标识上传给服务端，让服务端根据唯一标识判断对应的是哪种行为。这么做看起来是极其繁琐的，似乎看不到任何优点。缺点太明显容易掩盖了优点，先来看看这么做相比代码埋点的优点：

> 不用手写埋点代码了，但这还不是最重要的。最重要的是，这么做实现了埋点的实时更新，可以在app发布后手动更改埋点配置文件，就能对埋点进行修改。而不会因为仅仅是埋点错埋或者漏埋，就要发一个新版本

那么针对这种方式埋点又有什么可以优化的地方呢？

#### 3.2.1 减少无效请求

我们现在是全埋的方式，然后每个view的点击事件都发送给服务端一个请求。那么其实我们可以在app首次下载的时候，向服务端获取一个配置文件，然后缓存在本地。当然这个配置文件也是有版本号的，只要版本号与服务端保持一致，我们就一直用本地缓存的这份配置文件。这样一来，我们就可以在本地进行比对，判断该点击事件是否存在埋点，如果存在，才向服务器发送请求。

#### 3.2.2 可视化操作

现在最让我们头疼的，还是手动配置这个任务。

> 无埋点这个说法其实并不准确，我认为应该称之为可视化埋点

我们可以将这个配置的过程，转变成一种更友好的方式，那就是可视化操作。具体的做法就是：

（1）新增一个app编译版本，可以称之为运维版

（2）在运维版中，我们可以点击一个view，然后弹出一个对话框，可以输入业务ID和业务名称，然后点击保存就可以保存到一个配置文件中。已经配置的埋点，可以给操作者提供一种反馈方式，比如我现在处理的方式是给点击的view加一个红色外边框。

（3）全部配置完成后，可以上传整份配置文件给服务端（上传成功后的埋点样式为绿色外边框）

这样就实现了可视化埋点操作。这种埋点方式还有一个优点就是，将业务逻辑和数据采集分析分开了。这样一来，开发人员只用关心业务逻辑，运维人员去管理埋点配置。

## 四、无埋点无法解决的问题

无埋点只能根据view的唯一标识去配置文件中找到已配置的内容，或者获取当前view中的属性（能通过editText.getText().toString这种方法获取用户输入的数据），亦或是调用代码中存在的一些全局静态方法来获取数据。所以，无埋点方案无法获取某些运行时数据（比如实时网络请求回来数据）。

所以，埋点依然是必不可少的，无埋点解决的只是无需传参，只需要知道事件名称的用户行为。
