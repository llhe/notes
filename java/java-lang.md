Java notes
==================

1. java对象初始化顺序: 基类初始化(可以调用虚函数)，本身类的成员初始化(声明时候赋值操作)，本身构造函数体
3. String基本类型转换

    ```Java
    String a = String.valueOf(2);
    int i = Integer.parseInt(a);
    ```
4. 尽量使用`System.arraycopy()`代替通过来循环复制数组

    ```Java
public static void arraycopy(Object src,
                             int srcPos,
                             Object dest,
                             int destPos,
                             int length);
    ```
5. equals注意事项
    * 签名要正确  

    ```Java
    public boolean equals(Object other); // 注意参数，使用@Overide是个好习惯
    ```
  * equals与hashCode/equalsTo要保持一致
6. Java中获取文件名，行号，不同于C的预编译

    ```Java
    Thread.currentThread().getStackTrace()[2].getFileName();
    ```
    ```Java
    Thread.currentThread().getStackTrace()[2].getLineNumber();
    ```
7. `volatile`(>5.0): 保证了内存序，即读写会加入内存屏障，可用做多线程同步(example?)
71. `synchronized`: mutex，针对的是对象，不能为null和原子类型
8. 反射执行有一定开销，其中method查找开销较大，执行额外开销较小，所以尽可能缓存创建出来的method对象
9. 逃逸分析：如果对象未被外部引用，会在栈上分配，而不是堆上，避免gc
10. if(isDebug) {log.debug(“”)}的作用，因为Java没有预编译，debug函数参数的计算以及调用均有开销
11. StringBuilder/StringBuffer, 且在开始指定最大容量，避免内存分配。StringBuffer为线程安全的，性能稍慢
12. for(int i=0,len=list.size();i<len;i++)
13. 性能的critical path避免使用异常处理逻辑 -- 异常是为异常的情况而设计的，使用时也应该牢记这一原则。
创建异常开销很大，主要用于捕获所有栈帧，throw/try/catch相对开销很小
14. 复制哈希时在创建显得哈希表时应指定capacity大小，避免复制过程中分配内存，同ArrayList  

    ```Java
HashMap(int initialCapacity);
          // Constructs an empty HashMap with the specified initial capacity and the default load factor (0.75).
    ```
    ```Java
HashMap(int initialCapacity, float loadFactor);
          // Constructs an empty HashMap with the specified initial capacity and load factor.
    ```
15. final, finally, finalize
  * final: 修饰类，变量
  * finally: 用于实现C++的RAAI
  * finalize: Object类定义了finalize(), 用于在GC时调用，可重载，用于释放资源等。
16. Collection, Collections:
  * interface Collection: 接口，为其他接口如Set，List等接口的父接口
  * Class Collections: Util classes
17. inner/outer/left/right join: 实际就是笛卡尔积(outter是原始乘积，left/outer/inner依次取出右null，左null，左右null的结果)，self join就是join self无关键词无特殊之处:  
    ```SQL
    select t1.f1, t2.f2
      from table1 as t1
      (inner/outter/left/right) join table2 as t2
        on t1.filed1 = t2.field
      where cond;
    ```
18. 如果可能，尽量定义final类：这样所有方法就是final的，编译器会尽量内联所有的方法
19. 线程安全且lazy的单例模式实现([link](http://en.wikipedia.org/wiki/Double-checked_locking)用到了`volatile`, `double-checked locking`, 以及`inne class lazy initialization`特性):  
    ```Java
    // Works with acquire/release semantics for volatile
    // Broken under Java 1.4 and earlier semantics for volatile
    class Foo {
        private volatile Helper helper = null;
        public Helper getHelper() {
            Helper result = helper;
            if (result == null) {
                synchronized(this) {
                    result = helper;
                    if (result == null) {
                        helper = result = new Helper();
                    }
                }
            }
            return result;
        }
 
        // other functions and members...
    }
    ```
    ```Java
    // Correct lazy initialization in Java 
    @ThreadSafe
    class Foo {
        private static class HelperHolder {
           public static Helper helper = new Helper();
        }
 
        public static Helper getHelper() {
            return HelperHolder.helper;
        }
    }
    ```
20. JDBC操作 [link](http://coolshell.cn/articles/889.html)  
    ```JAVA
    public class OracleJdbcTest  
    {  
        String driverClass = "oracle.jdbc.driver.OracleDriver";  
  
        Connection con;  
  
        public void init(FileInputStream fs)
            throws ClassNotFoundException, SQLException, FileNotFoundException, IOException  
        {  
            Properties props = new Properties();  
            props.load(fs);  
            String url = props.getProperty("db.url");  
            String userName = props.getProperty("db.user");  
            String password = props.getProperty("db.password");
            // Driver类中含有一段静态初始化代码，在ClassLoader加载类时执行，会到DriverManager进行注册
            Class.forName(driverClass);
  
            con=DriverManager.getConnection(url, userName, password);  
        }  
  
        public void fetch() throws SQLException, IOException  
        {  
            PreparedStatement ps = con.prepareStatement("select SYSDATE from dual");  
            ResultSet rs = ps.executeQuery();  
  
            while (rs.next())  
            {  
                // do the thing you do  
            }  
            rs.close();  
            ps.close();  
        }  
  
        public static void main(String[] args)  
        {  
            OracleJdbcTest test = new OracleJdbcTest();
            test.init();  
            test.fetch();  
        }  
    }
    ```
21. Java方法签名包含返回值，如果呗调用的方法(如在不同的jar中)返回值变化则会`java.lang.NoSuchMethodError`。原因：1) 返回值设计调用ABI 2) 保证质量 (C++有mangling，C ABI？ platform specific)
22. getCanonicalName(), getName(), getSimpleName(): 对于非匿名类，第一种是所有部分都是.来连接，第二种内部类使用$，第三种只有名字部分。其中2是load class所使用的。
23. Java Serializable序列化：如果期望更多控制，可以重载`private void writeObject(ObjectOutputStream oos)`和`private void readObject(ObjectInputStream ois)`，一般做法是将需要自己序列化的域声明为`transient`：
    ```Java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer.
     */
    private transient Object[] elementData;

    // ...

    /**
     * Save the state of the <tt>ArrayList</tt> instance to a stream (that
     * is, serialize it).
     *
     * @serialData The length of the array backing the <tt>ArrayList</tt>
     *             instance is emitted (int), followed by all of its elements
     *             (each an <tt>Object</tt>) in the proper order.
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
	// Write out element count, and any hidden stuff
	int expectedModCount = modCount;
	s.defaultWriteObject();

        // Write out array length
        s.writeInt(elementData.length);

	// Write out all elements in the proper order.
	for (int i=0; i<size; i++)
            s.writeObject(elementData[i]);

	if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

    }

    /**
     * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
     * deserialize it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
	// Read in size, and any hidden stuff
	s.defaultReadObject();

        // Read in array length and allocate array
        int arrayLength = s.readInt();
        Object[] a = elementData = new Object[arrayLength];

	// Read in all elements in the proper order.
	for (int i=0; i<size; i++)
            a[i] = s.readObject();
    }
}
    ```
