JVM相关
===
类加载
* 类加载过程
    + 加载：将.class文件加载到方法区（还包括jsp等转换），在方法区生成一个Class对象
    + 链接    
        - 验证：确保class文件符合jvm规范
        - 准备：为类变量申请内存，并设置初始值（这个初始值现在还不是我们设置的值）
        - 解析：将符号引用（class文件里）转换成直接引用（内存里）
    + 初始化：执行类构造方法（<client>）的过程，先执行父类的初始化
	- 以下情况不触发类的初始化：
	    1. 引用静态字段只触发对应类的初始化，父类静态字段不会触发子类的初始化
	    2. 定义对象数组不触发类的初始化
	    3. 通过类名获取Class对象
	    4. Class.forName的initialize=false时
	- 触发实验
* 类加载器
    + 种类：
	- Bootstrap ClassLoader：负责加载JAVA_HOME/lib下的类
        - Extension ClassLoader：负责加载JAVA_HOME/lib/ext下的类
	- Application ClassLoader：负责加载自定义的类
        - 自己继承Application ClassLoader实现的类加载器
    + 双亲委派模型：调用类加载器的加载方法，首先调用父类的加载方法
