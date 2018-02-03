# 深入理解java虚拟机 摘要 #
***

## 一、自动内存管理机制 ##
***
### 3. 实战：OutOfMemoryError异常 ###

* Java堆溢出:

    >**测试代码:**
    >```
    >public class Tests {
    >    static class Obj{
    >    }
    >
    >    public static  void  main(String[] args) throws >Exception   {
    >        List<Obj> list = new ArrayList<>();
    >        while (true){
    >            list.add(new Obj());
    >        }
    >    }
    >}   
    >```
    >**启动参数:**
    >```
    >-Xmx20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
    >```
    >将堆的最小值-Xms参数与最大值-Xmx参数设置为一样即可避免堆自动扩展，通过参数-XX：+HeapDumpOnOutOfMemoryError可以让虚拟机在出现内存溢
    出异常时Dump出当前的内存堆转储快照以便事后进行分析

    >要解决这个区域的异常，一般的手段是先通过内存映像分析工具
    >**ideal可使用JProfiler和JMeter插件进行分析**

* 虚拟机栈和本地方法栈溢出
    >**测试代码:**
    >```
    >public static class JavaVMStackSOF {
    >    private int stackLength = 1;
    >
    >    public void stackLeak() {
    >        stackLength++;
    >        stackLeak();
    >    }
    >}
    >
    >    public static void main(String[]args)throws Throwable{
    >        JavaVMStackSOF oom=new JavaVMStackSOF();
    >        try{
    >            oom.stackLeak();
    >        }catch(Throwable e){
    >            System.out.println("stack length："+oom.stackLength);
    >            throw e;
    >        }
    >    }
    >```
    
    >**启动参数:**
    >```
    >-Xss20m
    >```

    >对于HotSpot来说，虽然-Xoss参数（设置本地方法栈大小）存在，但实际上是无效的，栈容量只由-Xss参数设定
    >如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出StackOverflowError异常。

    >如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出OutOfMemoryError异常。

    >这里把异常分成两种情况，看似更加严谨，但却存在着一些互相重叠的地方：当栈空间无法继续分配时，到底是内存太小，还是已使用的栈空间太大，其本质上只是对同一件事情的两种描述而已。

    > 在单个线程下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常。

* 本机直接内存溢出
    >DirectMemory容量可通过-XX：MaxDirectMemorySize指定，如果不指定，则默认与Java堆最大值（-Xmx指定）一样

    >**测试代码:**
    >```
    >public class Tests {
    >   private static final Long MB=1024L*1024L;
    >   public static void main(String[]args)throws Exception{
    >       Field unsafeField=Unsafe.class.getDeclaredFields()[0];
    >       unsafeField.setAccessible(true);
    >       Unsafe unsafe=(Unsafe)unsafeField.get(null);
    >       while(true){
    >           unsafe.allocateMemory(MB);
    >       }
    >   }
    >}
    >```
    
    >**启动参数:**
    >```
    >-Xss20m
    >```

    >**代码解释:**
    >越过了DirectByteBuffer类，直接通过反射获取Unsafe实例进行内存分配（Unsafe类的getUnsafe（）方法限制了只有引导类加载器才会返回实例，也就是设计者希望只有rt.jar中的类才能使用Unsafe的功能）。因为，虽然使用DirectByteBuffer分配内存也会抛出内存溢出异常，但它抛出异常时并没有真正向操作系统申请分配内存，而是通过计算得知内存无法分配，于是手动抛出异常，真正申请分配内存的方法是unsafe.allocateMemory（）。

    >**由DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，如果读者发现OOM之后Dump文件很小，而程序中又直接或间接使用了NIO，那就可以考虑检查一下是不是这方面的原因。**