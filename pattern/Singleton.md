1.单例模式的特点：

(1)无参构造函数要定义为private类型，这样可以保证在类的外部不能实例化

(2)通过一个静态方法或枚举获取对象实例

(3)要保证单例类不能通过反序列化构造多个对象

(4)要保证单例类在多线程情况下是线程安全的

2.单例模式的优点

(1)单例模式只生成一个实例，减少了系统性能开销

3.单例模式的缺点

(1)单例模式没有抽象层，扩展比较困难

(2)如果实例对象长时间没有被使用，可能会被回收掉

(3)单例类的职责过重，在一定程度上违背了单一职责原则

常见的单例模式实现方式

(1)饿汉式

(2)懒汉式

(3)双重检测锁

(4)静态内部类

(5)枚举式

我们首先看一下饿汉式的实现方式

    public class Singleton {

        private static Singleton singleton = new Singleton();

        private Singleton() {
        }

        public static Singleton getInstance() {
            return singleton;
        }
    }

特点：线程安全，类加载时就被实例化，但是不能实现延迟加载，但存在反射和反序列化漏洞(需要实现Serializable接口)

    static public void main(String[] args) throws Exception {
        //通过反射的方式生成多个对象
        Class clazz = Class.forName("com.pattern.singleton.Singleton");
        Constructor<Singleton> constructor = clazz.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton singleton = constructor.newInstance();
        Singleton singleton1 = constructor.newInstance();
        System.out.println(singleton == singleton1);//false
    }

如何防止反射的漏洞呢？

    public class Singleton {

        private static Singleton singleton = new Singleton();

        private Singleton() {
            //防止反射漏洞
            if (null != singleton) {
                throw new RuntimeException();
            }
        }

        public static Singleton getInstance() {
            return singleton;
        }
    }
    
测试反序列化漏洞

    static public void main(String[] args) throws Exception {
        Singleton s1  = Singleton.getInstance();
        FileOutputStream fos = new FileOutputStream("D:/singleton.txt");
        ObjectOutputStream oos = new ObjectOutputStream(fos);
        oos.writeObject(s1);
        oos.close();
        fos.close();

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("D:/singleton.txt"));
        Singleton s2 = (Singleton) ois.readObject();
        ois.close();
        System.out.println(s1 == s2);
    }

那又该如何防止反序列化呢？

    public class Singleton implements Serializable {

        private static Singleton singleton = new Singleton();

        private Singleton() {
        }

        public static Singleton getInstance() {
            return singleton;
        }

        //防止反序列化漏洞
        private Object readResolve() {
            return singleton;
        }
       }

优化后的代码

    public class Singleton implements Serializable {

        private static Singleton singleton = new Singleton();

        private Singleton() {
            if (singleton != null) {
                throw new RuntimeException();
            }
        }

        public static Singleton getInstance() {
            return singleton;
        }

        //防止反序列化漏洞
        private Object readResolve() {
            return singleton;
        }
    }

下面看一下懒汉式的实现方式

    public class Singleton implements Serializable {

        private static Singleton singleton;

        private Singleton() {
        }

        public static Singleton getInstance() {
            if (null == singleton) {
                singleton = new Singleton();
            }
            return singleton;
        }

        //防止反序列化漏洞
        private Object readResolve() {
            return singleton;
        }
    }
    
特点：实现了延迟加载，但是在多线程情况下有问题，没有防止反射生成多个对象

    public class Singleton implements Serializable {

        private static Singleton singleton;

        private Singleton() {
        }

        public static synchronized Singleton getInstance() {
            if (null == singleton) {
                singleton = new Singleton();
            }
            return singleton;
        }

        //防止反序列化漏洞
        private Object readResolve() {
            return singleton;
        }
    }
    
在getInstance()加上了synchronized加锁，防止了多线程问题，但是每次调用都加锁，又可能会带来性能问题(反射暂时还没解决)

下面看一下解决多线程性能问题

    public class Singleton implements Serializable {

        private static Singleton singleton;

        private Singleton() {
        }

        public static Singleton getInstance() {
            if (null == singleton) {
                synchronized (Singleton.class) {
                    if (null == singleton) {
                        singleton = new Singleton();
                    }
                }
            }
            return singleton;
        }

        //防止反序列化漏洞
        private Object readResolve() {
            return singleton;
        }
    }

我们再看一下双重检测锁的实现方式

    public class Singleton implements Serializable {

           private static Singleton instance = null;

        private Singleton() {
        }

        public static Singleton getInstance() {
            if (instance == null) {
                Singleton singleton;
                synchronized(Singleton.class) {
                    singleton = instance;
                    if (singleton == null) {
                        synchronized(Singleton.class) {
                            if (singleton == null) {
                                singleton = new Singleton();
                            }
                        }
                        instance = singleton;
                    }
                }
            }
            return instance;
        }

        private Object readResolve() {
            return instance;
        }
    }

特点：由于编译器优化原因和JVM底层内部模型的原因，偶尔会出问题，不建议使用。没有解决反射生成多个对象的问题。

看一下静态内部类的实现方式

    public class Singleton implements Serializable {

        private Singleton() {
        }

        private static class SingletonInstance {
            private static final Singleton singleton = new Singleton();
        }

        public static Singleton getInstance() {
            return  SingletonInstance.singleton;
        }

        private Object readResolve() {
            return SingletonInstance.singleton;
        }
    }
    
特点：

(1)外部类没有static属性，不会像饿汉式那样立即加载对象

(2)只有真正调用getInstance()才会加载静态内部类。加载类时是线程安全的。singleton是static final类型，保证了内存中只有这样一个实例存在，而且只能被赋值一次，从而保证了线程安全

(3)具有并发高效调用和延迟加载的优势

还是可以利用反射漏洞生成多个对象。

我们最后看一下枚举实现的单例

    public enum Singleton {
        INSTANCE;
    }
很简单，只有三行代码。

特点：实现简单，枚举本身就是单例。由JVM从根本上提供保障，避免了通过反射和反序列化的漏洞。

缺点就是不能实现延迟加载。

个人推荐静态内部类的实现方法。
