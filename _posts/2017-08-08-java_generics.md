---
layout: post
title: java 泛型
categories: Java
description: java 泛型
keywords: java, 泛型,
---


参考文章：

http://www.cnblogs.com/lwbqqyumidi/p/3837629.html

http://blog.csdn.net/caihuangshi/article/details/51278793

http://www.cnblogs.com/lucky_dai/p/5589317.html

# 泛型的概念及好处

泛型，也称为“参数化类型”，可以简单的理解为将类型也作为一个变量参数，在使用/调用时传入具体的类型。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。在未引入泛型之前，都是通过对类型Object的引用来实现参数的“任意化”，这样的话会带来两个问题：
- 当使用这个对象的时候，每一次都需要强制转换为需要的类型
- 对于强制转换类型的错误，编译器不会产生任何错误提示

而泛型的好处就是在编译的时候会检查类型安全，所有的强制转换都是自动和隐式的，提高代码的重用率。
可以来看下面一个对比例子：
```
public class GenericTest {

    public static void main(String[] args) {
        List list = new ArrayList();
        list.add("qqyumidi");
        list.add("corn");
        list.add(100);

        for (int i = 0; i < list.size(); i++) {
            String name = (String) list.get(i); // 编译不会报错，运行到这里会报错
            System.out.println("name:" + name);
        }
    }
}
```
在例子中，定义了一个List类型集合，其默认类型为Object类型，所以在加入String类型后不小心加入了Integer类型，在编译阶段编译类型还是Object，强制转换并不会报“java.lang.ClassCastException”异常，而在运行时运行类型就变为Integer本身，然后就会报错了。

再看使用泛型以后：
```
public class GenericTest {

    public static void main(String[] args) {
        List<String> list = new ArrayList<String>();
        list.add("qqyumidi");
        list.add("corn");
        //list.add(100);   // 提示编译错误

        for (int i = 0; i < list.size(); i++) {
            String name = list.get(i); // 自动转换
            System.out.println("name:" + name);
        }
    }
}
```
在使用泛型以后，在添加Integer类型对象就会出现编译错误。因为通过List<String>将String类型传递给List，就限定了List只能添加String类型对象，而且不需要强制类型转换，集合记住了此元素的类型。

# 泛型类

泛型类就是具有一个或多个类型参数的类，如下实例代码：
```

public class Pair<T, U> {
    private T first;
    private U second;
 
    public Pair(T first, U second) {
        this.first = first;
        this.second = second;
    }
 
    public T getFirst() {
        return first;
    }
 
    public U getSecond() {
        return second;
    }
 
    public void setFirst(T newValue) {
        first = newValue;
    }
 
    public void setSecond(U newValue) {
        second = newValue;
    }
}
```
泛型类Pair的类型参数为T、U，放在类名后的尖括号中。这里的T即Type的首字母，代表类型的意思，常用的还有E（element）、K（key）、V（value）等。当然不用这些字母指代类型参数也完全可以。
实例化泛型类的时候，我们只需要把类型参数换成具体的类型即可，比如实例化一个Pair<T, U>类我们可以这样：
```
Pair<String, Integer> pair = new Pair<String, Integer>();
```

# 泛型方法
所谓泛型方法，就是带有类型参数的方法，它既可以定义在泛型类中，也可以定义在普通类中。

# 类型变量的限定

在有些情况下，泛型类或者泛型方法想要对自己的类型参数进一步加一些限制。比如，我们想要限定类型参数只能为某个类的子类或者只能为实现了某个接口的类。相关的语法如下：
<T extends BoundingType>（BoundingType是一个类或者接口）。其中的BoundingType可以多于1个，用“&”连接即可。

下面来介绍一个实例：
我们有一个抽象类：Person，其有两个子类Student和Staff，其中Person中有一个自定义类Info用于记录Person的信息，而Student和Staff中也继承了这个类并丰富了其信息。在获取这两个类的信息时，按照正常的写法，我们需要在每一个子类中分别重写获取信息函数。这里我们就使用泛型，并限定泛型的类型为实现了这个接口的类，这样的话，我们就不用分别重写对应信息的获取，如下代码：
Person
```
public abstract class Person<T extends Person.Info> {
    protected T mInfo;
    public Person(String name) {
        mInfo = creatInfo();
    }

    protected abstract T creatInfo();

    public T getInfo() {
        return mInfo;
    }

    public
    static class Info {
        String mName;
        public String toString() {
            return "Person(" + mName + ")";
        }
    }
}
```
可以看到，T限定为类的信息类型，也就是具体实现类中信息类型，而getInfo也就返回对应类型的info。

Student：
```
public class Student extends Person<Student.StudentInfo>{
    private StudentInfo mStudentInfo;

    public Student(String name, int uid){
        super(name);
        mInfo.mName = name;
        mInfo.mUid = uid;
    }

    public StudentInfo creatInfo() {
        return new StudentInfo();
    }

    static class StudentInfo extends Person.Info {
        int mUid;
        @Override
        public String toString() {
            return "Student(" + mName + ", " + mUid + ")";
        }
    }
}
```

Staff：
```
public class Staff extends Person<Staff.StaffInfo>{

    public Staff(String name, double price){
        super(name);
        mInfo.mName = name;
        mInfo.mPrice = price;
    }

    public StaffInfo creatInfo() {
        return new StaffInfo();
    }

    static class StaffInfo extends Person.Info {
        double mPrice;

        @Override
        public String toString() {
            return "Staff(" + mName + ", " + mPrice + ")";
        }
    }
}
```

使用：
```
public class MainActivity extends AppCompatActivity {
    private Button bt1;
    private Button bt2;

    private Student mStudent;
    private Staff mStaff;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        bt1 = (Button) findViewById(R.id.bt1);
        bt2 = (Button) findViewById(R.id.bt2);

        mStudent = new Student("chris",1);
        mStaff = new Staff("yecong",0.5);

        bt1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("chris","mStudent:"+mStudent.getInfo().toString());
            }
        });

        bt2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Log.d("chris","mStaff:"+mStaff.getInfo().toString());
            }
        });
    }
}
```
输出如下：
![](/images/posts/android/java_泛型.png)

可以看到，我们在子类中并为重写getInfo，但是通过泛型在父类实现中限定了对应实现类的信息类型，所以子类调用父类的该方法时，都获取到了对应子类的信息，提高了代码的重用率。
