Java Memory Model
==============
针对Java 5.0之后，JSR-133

happen-before
------------------
1. happen-before基本规则 (from JSL7)
  * If x and y are actions of the same thread and x comes before y in program order,
then hb(x, y).
  * There is a happens-before edge from the end of a constructor of an object to the
start of a finalizer (§12.6) for that object.
  * If an action x synchronizes-with a following action y, then we also have hb(x, y).
  * If hb(x, y) and hb(y, z), then hb(x, z)

2. 涉及同步的happen-before规则 (from JSL7)
  * An unlock on a monitor happens-before every subsequent lock on that monitor.
  * A write to a volatile field (§8.3.1.4) happens-before every subsequent read of
that field.
  * A call to start() on a thread happens-before any actions in the started thread.
  * All actions in a thread happen-before any other thread successfully returns from
a join() on that thread.
  * The default initialization of any object happens-before any other actions (other
than default-writes) of a program.

3. 参考资料
  * http://docs.oracle.com/javase/specs/jls/se7/jls7.pdf

volatile
-------------------
1. 根据以下两条规则
  * 单线程环境中，两条语句x, y的先后顺序保证hb(x, y)
  * volatile变量写入后，后续的读操作保证hb(w, r)
可以推断出volatile可以用作线程同步，即volatile的读写会保证编译器(含JIT)和CPU的LOAD/STORE内存屏障

2. 应用  
  * 共享数据:

      ```Java
           Value value = null;
           volatile bool flag = false;
      ```
  * Thread 1:

      ```Java
           value = calculateValue(); // a1
           flag = true;
      ```
  * Thread 2:

      ```Java
           if (flag) {
               useValue(value); // a2
           }
      ```  
  volatile能够保证hb(a1, a2)，即使value是非volatile对象

3. 参考资料
  * http://docs.oracle.com/javase/specs/jls/se7/jls7.pdf
  * http://jeremymanson.blogspot.com/2008/11/what-volatile-means-in-java.html
  * http://www.ibm.com/developerworks/library/j-jtp03304/
  * http://g.oswego.edu/dl/jmm/cookbook.html

final
-------------------
1. 语义 (from JSL7)
> Given a write w, a freeze f, an action a (that is not a read of a final field), a read
r1 of the final field frozen by f, and a read r2 such that hb(w, f), hb(f, a), mc(a, r1),
and dereferences(r1, r2), then when determining which values can be seen by r2,
we consider hb(w, r2). (This happens-before ordering does not transitively close
with other happens-before orderings.)

2. Java concurrency in practice 中的说明
> An object is considered to be completely initialized when its constructor finishes. A
thread that can only see a reference to an object after that object has been completely
initialized is guaranteed to see the correctly initialized values for that object's final
fields.  
The usage model for final fields is a simple one: Set the final fields for an
object in that object's constructor; and do not write a reference to the object being
constructed in a place where another thread can see it before the object's constructor
is finished. If this is followed, then when the object is seen by another thread, that
thread will always see the correctly constructed version of that object's final fields.
It will also see versions of any object or array referenced by those final fields that
are at least as up-to-date as the final fields are.  

3. developerworks文章说明 
> The new JMM also seeks to provide a new guarantee of initialization safety -- that as long as an object is properly constructed (meaning that a reference to the object is not published before the constructor has completed), then all threads will see the values for its final fields that were set in its constructor, regardless of whether or not synchronization is used to pass the reference from one thread to another. Further, any variables that can be reached through a final field of a properly constructed object, such as fields of an object referenced by a final field, are also guaranteed to be visible to other threads as well. *This means that if a final field contains a reference to, say, a LinkedList, in addition to the correct value of the reference being visible to other threads, also the contents of that LinkedList at construction time would be visible to other threads without synchronization*. The result is a significant strengthening of the meaning of final -- that final fields can be safely accessed without synchronization, and that compilers can assume that final fields will not change and can therefore optimize away multiple fetches.

4. The JSR-133 Cookbook for Compiler Writers 
>  1. A store of a final field (inside a constructor) and, if the field is a reference, any store that this final can reference, cannot be reordered with a subsequent store (outside that constructor) of the reference to the object holding that field into a variable accessible to other threads. For example, you cannot reorder  
  ```
      x.finalField = v; ... ; sharedRef = x;
  ```  
  This comes into play for example when inlining constructors, where "..." spans the logical end of the constructor. You cannot move stores of finals within constructors down below a store outside of the constructor that might make the object visible to other threads. (As seen below, this may also require issuing a barrier). Similarly, you cannot reorder either of the first two with the third assignment in:  
  ```
       v.afield = 1; x.finalField = v; ... ; sharedRef = x;
  ```  
>  2. The initial load (i.e., the very first encounter by a thread) of a final field cannot be reordered with the initial load of the reference to the object containing the final field. This comes into play in:  
  ```
      x = sharedRef; ... ; i = x.finalField;
  ```  
A compiler would never reorder these since they are dependent, but there can be consequences of this rule on some processors.

5. 中文解释  
从同步角度，final包含了volatile的所有语义，所以Java中不能同时使用这两个关键字。只要能保证在构造函数中不使this逃逸，则构造函数结束时(或者说程序获取其引用时)，其所有的final域均保证完成内存的Store (即使此域内部不是递归final的？但是这种情况意义不大，因为包含递归的非final域，说明可以被改变，仍有出现同步问题的风险)。

6. 注意  
标准只针对类成员的final修饰符和构造函数中的赋值，其余不能保证，例如不保证final局部变量的可见性。
7. 反射  
反射过程可以修改final域，应注意安全问题

7. 参考资料
  * http://docs.oracle.com/javase/specs/jls/se7/jls7.pdf
  * http://www.ibm.com/developerworks/library/j-jtp03304/

变量的安全发布总结
-------------------
Java concurrency in practice (3.5.3):
  * 在静态初始化函数中初始化一个对象引用
  * 将对象引用保存到volatile类型域或者AtomicReference对象中
  * 将对象引用保存到某个正确构造函数对象的final类型域中
  * 将对象引用保存到一个由锁保护的域中
  * 将对象引用保存到线程安全的容器中(实际上等同于上述的volatile, 锁规则)
  
