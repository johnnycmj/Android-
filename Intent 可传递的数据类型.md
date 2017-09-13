## Intent 可传递的数据类型

Intent 传递的数据，基本类型都可以传递，除此之外对象也可以传递，但是对象必须进行序列化，序列化有两种一种是实现Serializable 接口，一种是实现Parcelable接口.

我们Activity在传递数据的过程，这个数据是保存在缓冲区的，比如parcel对象在不同activity直接传递过程中保存在一个叫做“ Binder transaction buffe”的地方，既然是缓冲区，那肯定有大小限制，缓冲区大小是1M，**Android规定Intent传递的数据最大不能超过1M。**

![inetnt_01.png](http://upload-images.jianshu.io/upload_images/6356977-406cc3b6d0dd5b7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![intent_02.png](http://upload-images.jianshu.io/upload_images/6356977-2a66b07055f07d19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 基本类型传递

基本类型传递比较简单：

```
Intent intent = new Intent(InstanceStateActivity.this,StandardActivity.class);
intent.putExtra("test string","String data");
intent.putExtra("test int",100);
intent.putExtra("test long",1000L);
intent.putExtra("test float",1.5);
startActivity(intent);
```

接收代码：

```
        getIntent().getStringExtra("test string");
        getIntent().getIntExtra("test int",0);
        getIntent().getLongExtra("test long",0L);
        getIntent().getFloatExtra("test float",0);
```

###  Serializable传递

首先有一个实现Serializable的Bean。

```
public class Student implements Serializable {

    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

传递Serializable对象：

```
        Student student = new Student();
        student.setName("Jim");
        student.setAge(21);

        Intent intent = new Intent(InstanceStateActivity.this,StandardActivity.class);
        intent.putExtra("student",student);
        startActivity(intent);
```

接收：

```
getIntent().getSerializableExtra("student");
```

传递List对象：

```
        List<Student> stuList = new ArrayList<Student>();
        Student student = new Student();
        student.setName("Jim");
        student.setAge(21);
        stuList.add(student);

        Intent intent = new Intent(InstanceStateActivity.this,StandardActivity.class);
        //将List强制转换成Serializable，传递
        intent.putExtra("stuList",(Serializable)stuList);
        startActivity(intent);
```

接收：

```
List<Student> list = (List<Student>)getIntent().getSerializableExtra("stuList");
```



###  Parcelable传递

首先有一个实现Parcelable的Bean。

```
public class Student1 implements Parcelable {

    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.name);
        dest.writeInt(this.age);
    }

    public Student1() {
    }

    protected Student1(Parcel in) {
        this.name = in.readString();
        this.age = in.readInt();
    }

    public static final Parcelable.Creator<Student1> CREATOR = new Parcelable.Creator<Student1>() {
        @Override
        public Student1 createFromParcel(Parcel source) {
            return new Student1(source);
        }

        @Override
        public Student1[] newArray(int size) {
            return new Student1[size];
        }
    };
}
```

传递对象：

```
        Student1 student = new Student1();
        student.setName("Jim");
        student.setAge(21);

        Intent intent = new Intent(InstanceStateActivity.this,StandardActivity.class);
        intent.putExtra("student1",student);
        startActivity(intent);
```

接收：

```
getIntent().getParcelableExtra("student1");
```

除了这些之外，Intent还可以传递Bundle对象。

### Bundle传递

```
        List<Student1> stuList = new ArrayList<Student1>();
        Student1 student = new Student1();
        student.setName("Jim");
        student.setAge(21);

        stuList.add(student);

        Bundle bundle = new Bundle();
        bundle.putParcelable("stu",student);
        bundle.putParcelableArrayList("stuList", (ArrayList<? extends Parcelable>) stuList);

        Intent intent = new Intent(InstanceStateActivity.this,StandardActivity.class);
        intent.putExtras(bundle);
        startActivity(intent);
```

接收：

```
        getIntent().getExtras().getParcelable("stu");
        getIntent().getExtras().getParcelableArrayList("stuList");
```

