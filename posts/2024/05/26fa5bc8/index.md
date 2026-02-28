# Java 反射 问题

在前面介绍 [Java 反射](/posts/2017/05/fa358471/)一文中,提到过反射的弊端，本文基于此将展开介绍一些案例。

<!--more-->

## 不稳定的顺序

### 背景

在实现缓存能力时，直接将请求体转为 json_str, 然后将 MD5(json_str) 作为缓存的Key。在生产环境会发现相同的查询在某些节点计算出来的缓存key 并不相同。进一步通过日志打印发现，json_str中属性顺序不一致。

### 定位

由于对应的类没有设置 Jackson 序列化时的顺序，实际序列化时使用的顺序来自 Java 反射的顺序。大致的调用栈关系如下:

```
toString:136, BaseJsonNode (com.fasterxml.jackson.databind.node)
-> toString:136, BaseJsonNode (com.fasterxml.jackson.databind.node)
-> nodeToString:30, InternalNodeMapper (com.fasterxml.jackson.databind.node)
-> writeValueAsString:1086, ObjectWriter (com.fasterxml.jackson.databind)
-> _writeValueAndClose:1219, ObjectWriter (com.fasterxml.jackson.databind)
-> serialize:1518, ObjectWriter$Prefetch (com.fasterxml.jackson.databind)
-> serializeValue:319, DefaultSerializerProvider (com.fasterxml.jackson.databind.ser)
-> _serialize:480, DefaultSerializerProvider (com.fasterxml.jackson.databind.ser)
-> serialize:20, SerializableSerializer (com.fasterxml.jackson.databind.ser.std)
-> serialize:39, SerializableSerializer (com.fasterxml.jackson.databind.ser.std)
-> serialize:115, POJONode (com.fasterxml.jackson.databind.node)
-> defaultSerializeValue:1142, SerializerProvider (com.fasterxml.jackson.databind)
-> findTypedValueSerializer:822, SerializerProvider (com.fasterxml.jackson.databind)
  # 获取序列化器有缓存机制, 所以同一个JVM 进程内，Json 序列化结果是稳定的。
-> findValueSerializer:544, SerializerProvider (com.fasterxml.jackson.databind)
-> _createAndCacheUntypedSerializer:1443, SerializerProvider (com.fasterxml.jackson.databind)
-> createSerializer:173, BeanSerializerFactory (com.fasterxml.jackson.databind.ser)
-> _createSerializer2:224, BeanSerializerFactory (com.fasterxml.jackson.databind.ser)
-> findSerializerByAnnotations:391, BasicSerializerFactory (com.fasterxml.jackson.databind.ser)
-> findJsonValueAccessor:258, BasicBeanDescription (com.fasterxml.jackson.databind.introspect)
-> getJsonValueAccessor:270, POJOPropertiesCollector (com.fasterxml.jackson.databind.introspect)
-> collectAll:422, POJOPropertiesCollector (com.fasterxml.jackson.databind.introspect)
-> _addMethods:680, POJOPropertiesCollector (com.fasterxml.jackson.databind.introspect)
  # _addMethods方法内会通过反射获取类的所有方法
```
> 在前面的文章中提到通过Java反射获取字段，方法和构造器时，返回结果的顺序是不稳定的(没有排序，且不保证一定按某种顺序)。

### 原因

`java.lang.Class.getDeclaredMethods0`方法调用了 native 方法`JVM_GetClassDeclaredMethods`,然后 native 方法又调用了`get_class_declared_methods_helper`方法。

```cpp
// https://github.com/openjdk/jdk/blob/master/src/hotspot/share/prims/jvm.cpp

static jobjectArray get_class_declared_methods_helper(
                                  JNIEnv *env,
                                  jclass ofClass, jboolean publicOnly,
                                  bool want_constructor,
                                  Klass* klass, TRAPS) {

  JvmtiVMObjectAllocEventCollector oam;

  oop ofMirror = JNIHandles::resolve_non_null(ofClass);
  // Exclude primitive types and array types
  if (java_lang_Class::is_primitive(ofMirror)
      || java_lang_Class::as_Klass(ofMirror)->is_array_klass()) {
    // Return empty array
    oop res = oopFactory::new_objArray(klass, 0, CHECK_NULL);
    return (jobjectArray) JNIHandles::make_local(THREAD, res);
  }

  InstanceKlass* k = InstanceKlass::cast(java_lang_Class::as_Klass(ofMirror));

  // Ensure class is linked
  k->link_class(CHECK_NULL);

  Array<Method*>* methods = k->methods();
  int methods_length = methods->length();

  // Save original method_idnum in case of redefinition, which can change
  // the idnum of obsolete methods.  The new method will have the same idnum
  // but if we refresh the methods array, the counts will be wrong.
  ResourceMark rm(THREAD);
  GrowableArray<int>* idnums = new GrowableArray<int>(methods_length);
  int num_methods = 0;

  // Select methods matching the criteria.
  for (int i = 0; i < methods_length; i++) {
    Method* method = methods->at(i);
    if (want_constructor && !method->is_object_initializer()) {
      continue;
    }
    if (!want_constructor &&
        (method->is_object_initializer() || method->is_static_initializer() ||
         method->is_overpass())) {
      continue;
    }
    if (publicOnly && !method->is_public()) {
      continue;
    }
    idnums->push(method->method_idnum());
    ++num_methods;
  }
  // Allocate result
  objArrayOop r = oopFactory::new_objArray(klass, num_methods, CHECK_NULL);
  objArrayHandle result (THREAD, r);

  // Now just put the methods that we selected above, but go by their idnum
  // in case of redefinition.  The methods can be redefined at any safepoint,
  // so above when allocating the oop array and below when creating reflect
  // objects.

  /// !!! 注意此处获取所有的方法
  for (int i = 0; i < num_methods; i++) {
    methodHandle method(THREAD, k->method_with_idnum(idnums->at(i)));
    if (method.is_null()) {
      // Method may have been deleted and seems this API can handle null
      // Otherwise should probably put a method that throws NSME
      result->obj_at_put(i, nullptr);
    } else {
      oop m;
      if (want_constructor) {
        m = Reflection::new_constructor(method, CHECK_NULL);
      } else {
        m = Reflection::new_method(method, false, CHECK_NULL);
      }
      result->obj_at_put(i, m);
    }
  }

  return (jobjectArray) JNIHandles::make_local(THREAD, result());
}
```

get_class_declared_methods_helper 方法中通过`k->method_with_idnum(idnums->at(i))`获取所有方法。下面我们关注下method_with_idnum 方法中的实现:

```cpp
// https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/instanceKlass.cpp
Method* InstanceKlass::method_with_idnum(int idnum) const {
  Method* m = nullptr;
  if (idnum < methods()->length()) {
    m = methods()->at(idnum);
  }
  if (m == nullptr || m->method_idnum() != idnum) {
    for (int index = 0; index < methods()->length(); ++index) {
      m = methods()->at(index);
      if (m->method_idnum() == idnum) {
        return m;
      }
    }
    // None found, return null for the caller to handle.
    return nullptr;
  }
  return m;
}
```
InstanceKlass里的methods是关键，而这个methods的创建是在类解析的时候发生的。

```cpp
// https://github.com/openjdk/jdk/blob/master/src/hotspot/share/classfile/classFileParser.cpp

// 首先通过 parse_stream -> parse_methods 解析出 _methods
parse_methods(stream,
                _access_flags.is_interface(),
                &_has_localvariable_table,
                &_has_final_method,
                &_declares_nonstatic_concrete_methods,
                CHECK);

assert(_methods != nullptr, "invariant");

// 在 parse_stream 完成后，通过 post_process_parsed_stream -> sort_methods对_methods进行排序
_method_ordering = sort_methods(_methods);
```

```cpp
// https://github.com/openjdk/jdk/blob/master/src/hotspot/share/classfile/classFileParser.cpp
static const intArray* sort_methods(Array<Method*>* methods) {
  const int length = methods->length();
  // If JVMTI original method ordering or sharing is enabled we have to
  // remember the original class file ordering.
  // We temporarily use the vtable_index field in the Method* to store the
  // class file index, so we can read in after calling qsort.
  // Put the method ordering in the shared archive.
  if (JvmtiExport::can_maintain_original_method_order() || CDSConfig::is_dumping_archive()) {
    for (int index = 0; index < length; index++) {
      Method* const m = methods->at(index);
      assert(!m->valid_vtable_index(), "vtable index should not be set");
      m->set_vtable_index(index);
    }
  }
  // Sort method array by ascending method name (for faster lookups & vtable construction)
  // Note that the ordering is not alphabetical, see Symbol::fast_compare
  Method::sort_methods(methods);

  intArray* method_ordering = nullptr;
  // If JVMTI original method ordering or sharing is enabled construct int
  // array remembering the original ordering
  if (JvmtiExport::can_maintain_original_method_order() || CDSConfig::is_dumping_archive()) {
    method_ordering = new intArray(length, length, -1);
    for (int index = 0; index < length; index++) {
      Method* const m = methods->at(index);
      const int old_index = m->vtable_index();
      assert(old_index >= 0 && old_index < length, "invalid method index");
      method_ordering->at_put(index, old_index);
      m->set_vtable_index(Method::invalid_vtable_index);
    }
  }
  return method_ordering;
}
```
从上面的`Method::sort_methods`可以看出比较的是两个方法的名字，但是这个名字不是一个字符串，而是一个Symbol对象，每个类或者方法名字都会对应一个Symbol对象，在这个名字第一次使用的时候构建，并且不是在java heap里分配的，比如jdk7里就是在c heap里通过malloc来分配的，jdk8里会在metaspace里分配.

```cpp
//https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/method.cpp
// This is only done during class loading, so it is OK to assume method_idnum matches the methods() array
// default_methods also uses this without the ordering for fast find_method
void Method::sort_methods(Array<Method*>* methods, bool set_idnums, method_comparator_func func) {
  int length = methods->length();
  if (length > 1) {
    if (func == nullptr) {
      func = method_comparator;
    }
    {
      NoSafepointVerifier nsv;
      QuickSort::sort(methods->data(), length, func);
    }
    // Reset method ordering
    if (set_idnums) {
      for (u2 i = 0; i < length; i++) {
        Method* m = methods->at(i);
        m->set_method_idnum(i);
        m->set_orig_method_idnum(i);
      }
    }
  }
}
// Comparer for sorting an object array containing
// Method*s.
static int method_comparator(Method* a, Method* b) {
  return a->name()->fast_compare(b->name());
}
// Note: this comparison is used for vtable sorting only; it doesn't matter
// what order it defines, as long as it is a total, time-invariant order
// Since Symbol*s are in C_HEAP, their relative order in memory never changes,
// so use address comparison for speed
int Symbol::fast_compare(const Symbol* other) const {
 return (((uintptr_t)this < (uintptr_t)other) ? -1
   : ((uintptr_t)this == (uintptr_t) other) ? 0 : 1);
}
```

从上面的fast_compare方法知道，其实对比的是地址的大小，因为Symbol对象是通过malloc来分配的，因此新分配的Symbol对象的地址就不一定比后分配的Symbol对象地址小，也不一定大，因为期间存在内存free的动作，那地址是不会一直线性变化的，之所以不按照字母排序，主要还是为了速度考虑，根据地址排序是最快的。

综上所述，一个类里的方法经过排序之后，顺序可能会不一样，取决于方法名对应的Symbol对象的地址的先后顺序。

### 影响

对于上面背景中提到的问题，造成的影响仅是缓存的错误刷新。但是实际上还会有更严重的影响，例如安全问题 [从JSON1链中学习处理JACKSON链的不稳定性](https://xz.aliyun.com/news/12292)。

> 上面的安全问题可以使用 [用于生成利用不安全Java对象反序列化的有效负载的概念验证工具](https://github.com/frohoff/ysoserial)进行验证。

## 内存问题

### 背景

服务FullGC次数超出阈值。

- 通过GC日志分析，发现是Metaspace 导致。
- 通过内存 dump，发现内存中出现较多的`sun.reflect.DelegatingClassLoader`和`sun.reflect.GeneratedMethodAccessorXXXX`。

### 定位

根据`sun.reflect.DelegatingClassLoader`和`sun.reflect.GeneratedMethodAccessorXXXX`基本可以确定是反射导致。所以我们可以直接从 JVM 中下载`GeneratedMethodAccessorXXXX` 文件来定位根因。

- 使用 [arthas dump 命令](https://arthas.aliyun.com/doc/dump.html#%E4%BD%BF%E7%94%A8%E5%8F%82%E8%80%83)导出 class.
- 使用 [jclasslib-bytecode-viewer](https://github.com/ingokegel/jclasslib)来阅读 class。

### 原理

先来看一个简单的例子:

```java
public class ReflectDemo {

    public static void main(String[] args) throws Exception{
        Proxy target = new ReflectDemo.Proxy();
        Method method = Proxy.class.getDeclaredMethod("pmethod", null);
        //MethodAccessor.invoke
        method.invoke(target, null);   
    }
    static class Proxy{
        public void pmethod(){
            System.out.println("Proxy.pmethod");
        }
    }
}
```

#### getDeclaredMethod

要调用首先要获取Method，而获取Method的逻辑是通过Class这个类来的。

在Class 类里有个关键的属性叫做reflectionData，这里主要存的是每次从jvm里获取到的一些类属性，比如方法，字段等。注意，这个属性主要是SoftReference的

```java
## java.lang.Class
private volatile transient SoftReference<ReflectionData<T>> reflectionData;
```

`getDeclaredMethod`方法其实很简单，就是从`privateGetDeclaredMethods`返回的方法列表里复制一个Method对象返回。而这个复制的过程是通过`searchMethods`实现的。

如果`reflectionData`这个属性的`declaredMethods`非空，那`privateGetDeclaredMethods`就直接返回其就可以了，否则就从JVM里去捞一把出来，并赋值给`reflectionData`的字段，这样下次再调用`privateGetDeclaredMethods`时候就可以用缓存数据了，不用每次调到JVM里去获取数据，因为`reflectionData`是`Softreference`，所以存在取不到值的风险，一旦取不到就又去JVM里捞了

`searchMethods`将从`privateGetDeclaredMethods`返回的方法列表里找到一个同名的匹配的方法，然后复制一个方法对象出来，这个复制的具体实现，其实就是Method.copy方法。

由此可见，我们每次通过调用`getDeclaredMethod`方法返回的Method对象其实都是一个新的对象，所以不宜多调，如果调用频繁最好缓存起来。

#### Method调用

接下来关注下 Method 的方法调用:

```java
public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, obj, modifiers);
        }
    }
    // read volatile
    MethodAccessor ma = methodAccessor;             
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}
```

可以看到`Method.invoke`方法就是调用`methodAccessor`的invoke方法。MethodAccessor本身是一个接口，有 3 个实现:

- DelegatingMethodAccessorImpl
- MethodAccessorImpl
- NativeMethodAccessorImpl

其中DelegatingMethodAccessorImpl是最终注入给Method的methodAccessor的，也就是某个Method的所有的invoke方法都会调用到这个DelegatingMethodAccessorImpl.invoke，正如其名一样的，是做代理的，也就是真正的实现是其他两种。

如果是NativeMethodAccessorImpl，那顾名思义，该实现主要是native实现的，而GeneratedMethodAccessorXXX是为每个需要反射调用的Method动态生成的类，后的XXX是一个数字，不断递增的。并且所有的方法反射都是先走NativeMethodAccessorImpl，默认调了15次之后，才生成一个GeneratedMethodAccessorXXX类，生成好之后就会走这个生成的类的invoke方法了。

那如何从NativeMethodAccessorImpl过度到GeneratedMethodAccessorXXX呢? 来看看NativeMethodAccessorImpl的invoke方法。

```java
// sun.reflect.NativeMethodAccessorImpl
public Object invoke(Object obj, Object[] args)
    throws IllegalArgumentException, InvocationTargetException
{
    // We can't inflate methods belonging to vm-anonymous classes because
    // that kind of class can't be referred to by name, hence can't be
    // found from the generated bytecode.
    if (++numInvocations > ReflectionFactory.inflationThreshold()
            && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
        MethodAccessorImpl acc = (MethodAccessorImpl)
            new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                                method.getName(),
                                method.getParameterTypes(),
                                method.getReturnType(),
                                method.getExceptionTypes(),
                                method.getModifiers());
        parent.setDelegate(acc);
    }

    return invoke0(method, obj, args);
}
```
> 上面说的是15次就是ReflectionFactory.inflationThreshold()这个方法返回的.由`-Dsun.reflect.inflationThreshold=xxx`来指定。我们还可以通过`-Dsun.reflect.noInflation=true`来直接绕过上面的15次NativeMethodAccessorImpl调用。

而`GeneratedMethodAccessorXXX`都是通过`new MethodAccessorGenerator().generateMethod`来生成的，一旦创建好之后就设置到`DelegatingMethodAccessorImpl`里去了，这样下次`Method.invoke`就会调到这个新创建的MethodAccessor里了。而生成的`GeneratedMethodAccessorXXX`里面就是直接调用目标对象的具体方法。

```java
// sun.reflect.ClassDefiner
static Class<?> defineClass(String name, byte[] bytes, int off, int len,
                            final ClassLoader parentClassLoader)
{
    ClassLoader newLoader = AccessController.doPrivileged(
        new PrivilegedAction<ClassLoader>() {
            public ClassLoader run() {
                    return new DelegatingClassLoader(parentClassLoader);
                }
            });
    return unsafe.defineClass(name, bytes, off, len, newLoader, null);
}
```

根据上面的逻辑，加载`GeneratedMethodAccessorXXX`的类加载器是一个DelegatingClassLoader类加载器。之所以搞一个新的类加载器，是为了性能考虑，在某些情况下可以卸载这些生成的类，因为类的卸载是只有在类加载器可以被回收的情况下才会被回收的，如果用了原来的类加载器，那可能导致这些新创建的类一直无法被卸载，从其设计来看本身就不希望他们一直存在内存里的，在需要的时候有就行了，在内存紧俏的时候可以释放掉内存。

NativeMethodAccessorImpl.invoke其实都是不加锁的，如果并发很高的时候，意味着可能同时有很多线程进入到创建GeneratedMethodAccessorXXX类的逻辑里，虽然说最终使用的其实只会有一个，但是这些开销已然存在了，假如有1000个线程都进入到创建GeneratedMethodAccessorXXX的逻辑里，那意味着多创建了999个无用的类，这些类会一直占着内存，直到Metaspace的GC发生才会回收。

## 参考

- [关于java反序列化中jackson链子不稳定问题](https://pankas.top/2023/10/04/%E5%85%B3%E4%BA%8Ejava%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%ADjackson%E9%93%BE%E5%AD%90%E4%B8%8D%E7%A8%B3%E5%AE%9A%E9%97%AE%E9%A2%98/)
- [JVM源码分析之不保证顺序的Class.getMethods](https://mp.weixin.qq.com/s/XrAD1Q09mJ-95OXI2KaS9Q)
- [假笨说-从一起GC血案谈到反射原理](https://mp.weixin.qq.com/s/5H6UHcP6kvR2X5hTj_SBjA)
- [反射代理类加载器的潜在内存使用问题](https://www.jianshu.com/p/20b7ab284c0a)

