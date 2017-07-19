---
layout: post
title: Android 序列化
categories: Android
description: Android 序列化
keywords: Andorid, Parcel, Serializable
---

####序列化的定义
Android序列化是指将一个实例对象编码成字节流（序列化），并从字节流编码中重新构建对象实例的能力（反序列化）。

####序列化的目的
1.永久的保存对象数据（将对象数据保存在文件或者磁盘中，此时对象是以一连串的字节形式保存）
2.通过序列化操作将对象数据在网络上进行传输（网络传输是以字节流的方式对数据进行传输）
3.对象数据在进程间传递（传递对象数据时，在一个进程中序列化，在另一个进程中反序列化），在Intent之间，基本的数据类型直接进行相关的传递，比较复杂的数据类型就需要序列化了。
>序列化对象只是针对变量进行，不针对方法序列化
序列化的一个简单应用：程序在断电或者异常终止，对象的工作状态也会丢失，此时使用序列化将对象的全部内同存于磁盘文件，就可以解决数据丢失的问题。

####序列化的方法
Android实现序列化有两种方式：Java Serializable和Parcelable。
首先来看Serializable，实现比较简单，只需要将类实现Serializable接口，就将该对象标注为可序列化。需要注意以下几点：
- serialVersionUID用来标识当前序列化对象的类版本，每一个实现Serializable的类都应该指定该域。如果没有指定，JVM会根据类的信息自动生成一个UID
- 被transient描述的域和类的静态变量不会被序列化，序列化只针对类实例
- 需要进行序列化的对象所有的域都必须实现Serializable接口，不然会直接报NotSerializableException，除非域为空或者被transient描述不会报错
- 如果实现了Serializable类的对象继承自另外一个类，那么这个类要么需要继承自Serializable，要么需要提供一个无参构造器

下面是一段代码实例：
```
public class Student extends Person implements Serializable {
    private static final long serialVersionUID = 1L;
    public String name;
    public static int static_field;
    public transient int transient_field;
    public Student(String name, int sex) {
        super(sex);
        this.name = name;
    }
    public void printLog() {
        Log.d("Chris","name:"+name+" static_field:"+static_field+" transient_field:"+transient_field+" sex:"+sex);
    }
}

public static void main(String[] args) throws IOException, ClassNotFoundException {
    Student mStudent = new Student("Chris", "男");
    mStudent.transient_field = 10;
    mStudent.static_field = 100;
    ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
    ObjectOutputStream outputStream = new ObjectOutputStream(byteArrayOutputStream);
    outputStream.writeObject(mStudent);
    outputStream.flush();
    outputStream.close();

    mStudent.static_field = 101;

    byte[] bytes = byteArrayOutputStream.toByteArray();
    ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
    ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
    Student mGetInfo = (Student) objectInputStream.readObject();
    mGetInfo.printLog();
}

class Person {
    public String sex;
    public Person() {}
    public Person(String sex) {
        this.sex = sex;
    }
}
```
输出结果为：
```
name:Chris static_field:101 transient_field:0 sex:null
```
> - ByteArrayOutputStream:此类实现了一个输出流，其中的数据被写入一个 byte 数组。缓冲区会随着数据的不断写入而自动增长。可使用 toByteArray()和 toString()获取数据。
- ByteArrayInputStream:包含一个内部缓冲区，该缓冲区包含从流中读取的字节,它需要提供一个byte数组作为缓冲区。
- ObjectInputStream和ObjectOutputStream类创建的对象被称为对象输入流和对象输出流。读取和写入分别为readObject、writeObject。

如果是通过Intent传递对象，则是如下：
封装：
Bundle.putSerializable(Key,Object);  //实现Serializable接口的对象
获取：
Intent.getSerializableExtra

下面来看另一种方式：Parcelable
Pracel提供了一套机制，可以将序列化后的数据写入到一个共享内存中，其它进程可以通过Parcel从这块共享内存中读出字节流，并反序列成对象。
我们先来看Parcelable接口的定义：
```
    public interface Parcelable {  
        //内容描述接口，基本不用管  
        public int describeContents();  
        //写入接口函数，打包  
        public void writeToParcel(Parcel dest, int flags);  
         //读取接口，目的是要从Parcel中构造一个实现了Parcelable的类的实例处理。因为实现类在这里还是不可知的，所以需要用到模板的方式，继承类名通过模板参数传入。  
        //为了能够实现模板参数的传入，这里定义Creator嵌入接口,内含两个接口函数分别返回单个和多个继承类实例。  
        public interface Creator<T> {  
               public T createFromParcel(Parcel source);  
               public T[] newArray(int size);  
        }  
```
所以要实现Parcelable，步骤如下：
1. implements Parcelable
2. 重写writeToParcel方法，将对象序列化成一个Parcel对象，即：将类的数据写入外部提供的Parcel中，打包需要传递的数据到Parcel容器保存，以便从Parcel容器中获取数据
3. 重写describeContents方法，内容接口描述，默认返回0就OK
4. 实例化静态内部对象CRATROE，实现接口Parcelable.Creator，其中createFromParcel的功能就是从Parcel读取类对象

>需要注意的是类里面的数据写的顺序和读的数据必须一致

下面是一个实例代码：
```
public class Student implements Parcelable {
    private String name;
    private int id;

    public Student(Parcel in) {
        this.name = in.readString();
        this.id = in.readInt();
    }

    public Student(String name, int id) {
        this.name = name;
        this.id = id;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(id);
    }

    public final static Parcelable.Creator<Student> CREATOR = new Parcelable.Creator<Student>() {
        @Override
        public Student createFromParcel(Parcel in) {
            return new Student(in);
        }

        @Override
        public Student[] newArray(int size) {
            return new Student[size];
        }
    };
}
```
通过writeToParcel将Student对象映射成Parcel对象，再通过creatFromParcel将Parcel对象映射成Student对象（其实可以将Parcel看成一个流）。
使用方法如下：
```
Parcel mParcel = Parcel.obtain
Student mStudent = new Student("Chris", 0);

//写入Parcel
mParcel.writeParcelable(mParcel,0);
//Parcel读写共用一个位置计数，这里一定要重置一下当前的位置
parcel.setDataPosition(0);

//读取Parcel
Student stu = parcel.readParcelable(Student.class.getClassLoader());
```
同样的，如果在Intent传递对象，则使用如下方法写入和获取：
封装：
Bundle.putParcelable(Key,Object);  //实现Parcelable接口的对象
获取：
Intent.getParcelableExtra

####两种方法的对比
1.在使用内存的时候Parcelable比Serializable的性能高。
2.Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC（内存回收）。
3.Parcelable不能使用在将对象存储在磁盘上这种情况，因为在外界的变化下Parcelable不能很  好的保证数据的持续性。
