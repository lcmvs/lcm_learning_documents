# lambda表达式

Lambda表达式是Java 8新引入的一种语法，是一种紧凑的传递代码的方式，它的名字来源于学术界的λ演算。

[深入探索Java 8 Lambda表达式](<https://droidyue.com/blog/2015/11/28/article-java-8-lambdas-a-peek-under-the-hood/>)

[Java 8里面lambda的最佳实践](<https://my.oschina.net/benhaile/blog/408377>)

## 通过接口传递代码

面向接口的编程，针对接口而非具体类型进行编程，可以降低程序的耦合性、提高灵活性、提高复用性。

接口常被用于传递代码：

```java
public String[] list(FilenameFilter filter)
public File[] listFiles(FilenameFilter filter)
```

list和listFiles需要的其实不是FilenameFilter对象，而是它包含的如下方法：

```java
boolean accept(File dir, String name);
```

或者说，list和listFiles希望接受一段方法代码作为参数，但没有办法直接传递这个方法代码本身，只能传递一个接口。

再比如，Collections的一些算法，很多方法都接受一个参数Comparator，比如：

```java
public static <T> int binarySearch(List<? extends T> list, T key, Comparator<? super T> c)
public static <T> T max(Collection<? extends T> coll, Comparator<? super T> comp)
public static <T> void sort(List<T> list, Comparator<? super T> c)
```

它们需要的也不是Comparator对象，而是它包含的如下方法：

```java
int compare(T o1, T o2);
```

但是，没有办法直接传递方法，只能传递一个接口。

异步任务执行服务ExecutorService，提交任务的方法有：

```java
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

Callable和Runnable接口也用于传递任务代码。

## 匿名内部类

通过接口传递行为代码，就要传递一个实现了该接口的实例对象，最简洁的方式是使用匿名内部类，比如：

```java
//列出当前目录下的所有后缀为.txt的文件
File f = new File(".");
File[] files = f.listFiles(new FilenameFilter(){
    @Override
    public boolean accept(File dir, String name) {
        if(name.endsWith(".txt")){
            return true;
        }
        return false;
    }
});
```

将files按照文件名排序，代码为：

```java
Arrays.sort(files, new Comparator<File>() {

    @Override
    public int compare(File f1, File f2) {
        return f1.getName().compareTo(f2.getName());
    }
});
```

提交一个最简单的任务，代码为：

```java
ExecutorService executor = Executors.newFixedThreadPool(100);
executor.submit(new Runnable() {
    @Override
    public void run() {
        System.out.println("hello world");
    }
});
```

## Lambda式

### 语法

Java 8提供了一种新的紧凑的传递代码的语法 - Lambda表达式。对于前面列出文件的例子，代码可以改为：

```java
File f = new File(".");
File[] files = f.listFiles((File dir, String name) -> {
    if (name.endsWith(".txt")) {
        return true;
    }
    return false;
});
```

可以看出，相比匿名内部类，传递代码变得更为直观，不再有实现接口的模板代码，不再声明方法，也名字也没有，而是直接给出了方法的实现代码。**Lambda表达式由->分隔为两部分，前面是方法的参数，后面{}内是方法的代码。**

上面代码可以简化为：

```java
File[] files = f.listFiles((File dir, String name) -> {
    return name.endsWith(".txt");
});
```

**当主体代码只有一条语句的时候，括号和return语句也可以省略，上面代码可以变为：**

```java
File[] files = f.listFiles((File dir, String name) -> name.endsWith(".txt"));
```

**注意，没有括号的时候，主体代码是一个表达式，这个表达式的值就是函数的返回值，结尾不能加分号;，也不能加return语句。**

**方法的参数类型声明也可以省略**，上面代码还可以继续简化为：

```java
File[] files = f.listFiles((dir, name) -> name.endsWith(".txt"));
```

之所以可以省略方法的参数类型，是因为Java可以自动推断出来，它知道listFiles接受的参数类型是FilenameFilter，这个接口只有一个方法accept，这个方法的两个参数类型分别是File和String。

这样简化下来，代码是不是简洁清楚多了?

排序的代码用Lambda表达式可以写为：

```java
Arrays.sort(files, (f1, f2) -> f1.getName().compareTo(f2.getName())); 
```

提交任务的代码用Lambda表达式可以写为：

```java
executor.submit(()->System.out.println("hello"));
```

**参数部分为空，写为()**。

**当参数只有一个的时候，参数部分的括号可以省略**，比如，File还有如下方法：

```java
public File[] listFiles(FileFilter filter)
```

FileFilter的定义为：

```java
public interface FileFilter {
    boolean accept(File pathname);
}
```

使用FileFilter重写上面的列举文件的例子，代码可以为：

```java
File[] files = f.listFiles(path -> path.getName().endsWith(".txt"));
```

### 变量引用

**与匿名内部类类似，Lambda表达式也可以访问定义在主体代码外部的变量，但对于局部变量，它也只能访问final类型的变量，与匿名内部类的区别是，它不要求变量声明为final，但变量事实上不能被重新赋值。**比如：

```java
String msg = "hello world";
executor.submit(()->System.out.println(msg));
```

可以访问局部变量msg，但msg不能被重新赋值，如果这样写：

```java
String msg = "hello world";
msg = "good morning";
executor.submit(()->System.out.println(msg));
```

Java编译器会提示错误。

这个原因与匿名内部类是一样的，Java会将msg的值作为参数传递给Lambda表达式，为Lambda表达式建立一个副本，它的代码访问的是这个副本，而不是外部声明的msg变量。如果允许msg被修改，则程序员可能会误以为Lambda表达式会读到修改后的值，引起更多的混淆。

为什么非要建副本，直接访问外部的msg变量不行吗？不行，因为msg定义在栈中，当Lambda表达式被执行的时候，msg可能早已被释放了。如果希望能够修改值，可以将变量定义为实例变量，或者，将变量定义为数组，比如：

```java
String[] msg = new String[]{"hello world"};
msg[0] = "good morning";
executor.submit(()->System.out.println(msg[0]));
```

### **与匿名内部类比较**

从以上内容可以看出，Lambda表达式与匿名内部类很像，主要就是简化了语法，那它是不是语法糖，内部实现其实就是内部类呢？答案是否定的，**Java会为每个匿名内部类生成一个类，但Lambda表达式不会**，Lambda表达式通常比较短，为每个表达式生成一个类会生成大量的类，性能会受到影响。

Java利用了Java 7引入的为支持动态类型语言引入的invokedynamic指令、方法句柄(method  handle)等，具体实现比较复杂，我们就不探讨了，感兴趣可以参看http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html，**我们需要知道的是，Java的实现是非常高效的，不用担心生成太多类的问题**。

Lambda表达式不是匿名内部类，那它的类型到底是什么呢？是函数式接口。

### **函数式接口**

Java 8引入了函数式接口的概念，**函数式接口也是接口，但只能有一个抽象方法**，前面提及的接口都只有一个抽象方法，都是函数式接口。之所以强调是"抽象"方法，是因为Java 8中还允许定义其他方法，我们待会会谈到。Lambda表达式可以赋值给函数式接口，比如：

```java
FileFilter filter = path -> path.getName().endsWith(".txt");
FilenameFilter fileNameFilter = (dir, name) -> name.endsWith(".txt");
Comparator<File> comparator = (f1, f2) -> f1.getName().compareTo(f2.getName());
Runnable task = () -> System.out.println("hello world");
```

如果看这些接口的定义，会发现它们都有一个注解@FunctionalInterface，比如：

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

@FunctionalInterface用于清晰地告知使用者，这是一个函数式接口，不过，这个注解不是必需的，不加，只要只有一个抽象方法，也是函数式接口。但如果加了，而又定义了超过一个抽象方法，Java编译器会报错，这类似于Override注解。



### 预定义的函数式接口

#### 接口列表

Java 8定义了大量的预定义函数式接口，用于常见类型的代码传递，这些函数定义在包java.util.function下，主要的有：

![img](assets/924211-20170705204539409-168614794.jpg)

对于基本类型boolean, int, long和double，为避免装箱/拆箱，Java 8提供了一些专门的函数，比如，int相关的主要函数有：

![img](assets/924211-20170705204612909-86398551.jpg)

这些函数有什么用呢？它们被大量使用于Java 8的函数式数据处理Stream相关的类中，关于Stream，我们下节介绍。

即使不使用Stream，也可以在自己的代码中直接使用这些预定义的函数，我们看一些简单的示例。

#### Predicate示例

为便于举例，我们先定义一个简单的学生类Student，有name和score两个属性，如下所示，我们省略了getter/setter方法。

```java
static class Student {
    String name;
    double score;
    
    public Student(String name, double score) {
        this.name = name;
        this.score = score;
    }
}
```

有一个学生列表:

```java
List<Student> students = Arrays.asList(new Student[] {
        new Student("zhangsan", 89d),
        new Student("lisi", 89d),
        new Student("wangwu", 98d) });
```

在日常开发中，列表处理的一个常见需求是过滤，列表的类型经常不一样，过滤的条件也经常变化，但主体逻辑都是类似的，可以借助Predicate写一个通用的方法，如下所示：

```java
public static <E> List<E> filter(List<E> list, Predicate<E> pred) {
    List<E> retList = new ArrayList<>();
    for (E e : list) {
        if (pred.test(e)) {
            retList.add(e);
        }
    }
    return retList;
}
```

这个方法可以这么用：

```java
// 过滤90分以上的
students = filter(students, t -> t.getScore() > 90);
```

#### Function示例

列表处理的另一个常见需求是转换，比如，给定一个学生列表，需要返回名称列表，或者将名称转换为大写返回，可以借助Function写一个通用的方法，如下所示：

```java
public static <T, R> List<R> map(List<T> list, Function<T, R> mapper) {
    List<R> retList = new ArrayList<>(list.size());
    for (T e : list) {
        retList.add(mapper.apply(e));
    }
    return retList;
}
```

根据学生列表返回名称列表的代码可以为：

```java
List<String> names = map(students, t -> t.getName());
```

将学生名称转换为大写的代码可以为：

```java
students = map(students, t -> new Student(t.getName().toUpperCase(), t.getScore()));
```

#### **Consumer示例**

在上面转换学生名称为大写的例子中，我们为每个学生创建了一个新的对象，另一种常见的情况是直接修改原对象，具体怎么修改通过代码传递，这时，可以用Consumer写一个通用的方法，比如：

```java
public static <E> void foreach(List<E> list, Consumer<E> consumer) {
    for (E e : list) {
        consumer.accept(e);
    }
}
```

上面转换为大写的例子可以改为：

```java
foreach(students, t -> t.setName(t.getName().toUpperCase()));
```

以上这些示例主要用于演示函数式接口的基本概念，实际中应该使用后面介绍的流API。

### **方法引用**

#### **基本用法**

Lambda表达式经常就是调用对象的某个方法，比如：

```java
List<String> names = map(students, t -> t.getName());
```

这时，它可以进一步简化，如下所示：

```java
List<String> names = map(students, Student::getName);
```

Student::getName这种写法，是Java 8引入的一种新语法，称之为方法引用，它是Lambda表达式的一种简写方法，由::分隔为两部分，**前面是类名或变量名**，**后面是方法名**。方法可以是实例方法，也可以是静态方法，但含义不同。

我们看一些例子，还是以Student为例，先增加一个静态方法：

```java
public static String getCollegeName(){
    return "Laoma School";
}
```

#### **静态方法**

对于静态方法，如下语句：

```java
Supplier<String> s = Student::getCollegeName;
```

等价于：

```java
Supplier<String> s = () -> Student.getCollegeName();
```

它们的参数都是空，返回类型为String。

#### **实例方法**

而对于实例方法，它第一个参数就是该类型的实例，比如，如下语句：

```java
Function<Student, String> f = Student::getName;
```

等价于：

```java
Function<Student, String> f = (Student t) -> t.getName();
```

对于Student::setName，它是一个BiConsumer，即：

```java
BiConsumer<Student, String> c = Student::setName;
```

等价于：

```java
BiConsumer<Student, String> c = (t, name) -> t.setName(name);
```

#### **通过变量引用方法**

如果方法引用的第一部分是变量名，则相当于调用那个对象的方法，比如：

```java
Student t = new Student("张三", 89d);
Supplier<String> s = t::getName;
```

等价于：

```java
Supplier<String> s = () -> t.getName(); 
```

而：

```java
Consumer<String> consumer = t::setName;
```

等价于：

```java
Consumer<String> consumer = (name) -> t.setName(name);
```

#### **构造方法**

对于构造方法，方法引用的语法是<类名>::new，如Student::new，如下语句：

```java
BiFunction<String, Double, Student> s = (name, score) -> new Student(name, score);
```

等价于：

```java
BiFunction<String, Double, Student> s = Student::new;
```

### **函数的复合**

在前面的例子中，函数式接口都用作方法的参数，其他部分通过Lambda表达式传递具体代码给它，函数式接口和Lambda表达式还可用作方法的返回值，传递代码回调用者，将这两种用法结合起来，可以构造复合的函数，使程序简洁易读。

下面我们会看一些例子，在介绍例子之前，我们先需要介绍Java 8对接口的增强。

#### **接口的静态方法和默认方法**

在Java 8之前，接口中的方法都是抽象方法，都没有实现体，Java 8允许在接口中定义两类新方法：静态方法和默认方法，它们有实现体，比如：


```java
public interface IDemo {
    void hello();

    public static void test() {
        System.out.println("hello");
    }

    default void hi() {
        System.out.println("hi");
    }
}
```


test()就是一个静态方法，可以通过IDemo.test()调用。在接口不能定义静态方法之前，相关的静态方法往往定义在单独的类中，比如，Collection接口有一个对应的单独的类Collections，在Java  8中，就可以直接写在接口中了，比如Comparator接口就定义了多个静态方法。

hi()是一个默认方法，由关键字default标识，默认方法与抽象方法都是接口的方法，不同在于，它有默认的实现，实现类可以改变它的实现，也可以不改变。引入默认方法主要是函数式数据处理的需求，是为了便于给接口增加功能。

在没有默认方法之前，Java是很难给接口增加功能的，比如List接口，因为有太多非Java  JDK控制的代码实现了该接口，如果给接口增加一个方法，则那些接口的实现就无法在新版Java  上运行，必须改写代码，实现新的方法，这显然是无法接受的。函数式数据处理需要给一些接口增加一些新的方法，所以就有了默认方法的概念，接口增加了新方法，而接口现有的实现类也不需要必须实现它。

看一些例子，List接口增加了sort方法，其定义为：


```java
default void sort(Comparator<? super E> c) {
    Object[] a = this.toArray();
    Arrays.sort(a, (Comparator) c);
    ListIterator<E> i = this.listIterator();
    for (Object e : a) {
        i.next();
        i.set((E) e);
    }
}
```


Collection接口增加了stream方法，其定义为：

```java
default Stream<E> stream() {
    return StreamSupport.stream(spliterator(), false);
}
```

需要说明的是，即使能定义方法体了，接口与抽象类还是不一样的，接口中不能定义实例变量，而抽象类可以。

了解了静态方法和默认方法，我们看一些利用它们实现复合函数的例子。

#### **Comparator中的复合方法**

Comparator接口定义了如下静态方法：


```java
public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
        Function<? super T, ? extends U> keyExtractor)
{
    Objects.requireNonNull(keyExtractor);
    return (Comparator<T> & Serializable)
        (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
}
```


这个方法是什么意思呢？它用于构建一个Comparator，比如，在前面的例子中，对文件按照文件名排序的代码为：

```java
Arrays.sort(files, (f1, f2) -> f1.getName().compareTo(f2.getName()));
```

使用comparing方法，代码可以简化为：

```java
Arrays.sort(files, Comparator.comparing(File::getName));
```

这样，代码的可读性是不是大大增强了？comparing方法为什么能达到这个效果呢？它构建并返回了一个符合Comparator接口的Lambda表达式，这个Comparator接受的参数类型是File，它使用了传递过来的函数代码keyExtractor将File转换为String进行比较。像comparing这样使用复合方式构建并传递代码并不容易阅读和理解，但调用者很方便，也很容易理解。

Comparator还有很多默认方法，我们看两个：


```java
default Comparator<T> reversed() {
    return Collections.reverseOrder(this);
}


default Comparator<T> thenComparing(Comparator<? super T> other) {
    Objects.requireNonNull(other);
    return (Comparator<T> & Serializable) (c1, c2) -> {
        int res = compare(c1, c2);
        return (res != 0) ? res : other.compare(c1, c2);
    };
}
```


reversed返回一个新的Comparator，按原排序逆序排。thenComparing也是一个返回一个新的Comparator，在原排序认为两个元素排序相同的时候，使用提供的other Comparator进行比较。

看一个使用的例子，将学生列表按照分数倒序排(高分在前)，分数一样的，按照名字进行排序，代码如下所示：

```java
students.sort(Comparator.comparing(Student::getScore)
                        .reversed()
                        .thenComparing(Student::getName));
```

这样，代码是不是很容易读？

#### **java.util.function中的复合方法**

在java.util.function包中的很多函数式接口里，都定义了一些复合方法，我们看一些例子。

Function接口有如下定义：

```java
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after);
    return (T t) -> after.apply(apply(t));
}
```

先将T类型的参数转化为类型R，再调用after将R转换为V，最后返回类型V。

还有如下定义：

```java
default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
    Objects.requireNonNull(before);
    return (V v) -> apply(before.apply(v));
}
```

对V类型的参数，先调用before将V转换为T类型，再调用当前的apply方法转换为R类型返回。

