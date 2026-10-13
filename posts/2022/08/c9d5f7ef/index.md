# Java异常丢失堆栈信息

在排查线上问题的时候，发现日志中只有`java.lang.NullPointerException: null`，没有打印出日志堆栈。

<!--more-->


第一反应是`log.error()`用错了方法。

```java
void error(String msg);
void error(String format, Object... arguments);
void error(String msg, Throwable t);
```

在打印异常的时候建议使用 `void error(String msg, Throwable t)`这个[方法](https://www.slf4j.org/faq.html#exception_message)。

如果想使用`void error(String format, Object... arguments)`这个[方法](https://www.slf4j.org/faq.html#paramException)，需要注意2点:

1. slf4j的版本要大于或等于1.6.0.
2. 使用占位符时，不要带上Exception。

```java
String s = "Hello world";
try {
  Integer i = Integer.valueOf(s);
} catch (NumberFormatException e) {
  logger.error("Failed to format {}", s, e);
}
```

仔细检查了代码，发现并不是这个问题引起的。

## OmitStackTraceInFastThrow

JVM中有个参数：OmitStackTraceInFastThrow，字面意思是省略异常栈信息从而快速抛出，那么JVM是如何做到快速抛出的呢？JVM对一些特定的异常类型做了Fast Throw优化，如果检测到在代码里某个位置连续多次抛出同一类型异常的话，C2会决定用Fast Throw方式来抛出异常，而异常Trace即详细的异常栈信息会被清空。

这种异常抛出速度非常快，因为不需要在堆里分配内存，也不需要构造完整的异常栈信息。该默认式在 -server 模式下是默认开启的。

OmitStackTraceInFastThrow 参数最早是在 [JDK5中引入](https://www.oracle.com/java/technologies/javase/release-notes-introduction.html)的。

> The compiler in the server VM now provides correct stack backtraces for all "cold" built-in exceptions. For performance purposes, when such an exception is thrown a few times, the method may be recompiled. After recompilation, the compiler may choose a faster tactic using preallocated exceptions that do not provide a stack trace. To disable completely the use of preallocated exceptions, use this new flag: -XX:-OmitStackTraceInFastThrow.


相关的源码的JVM源码的[graphKit.cpp文件](https://github.com/openjdk/jdk/blob/8153779ad32d1e8ddd37ced826c76c7aafc61894/hotspot/src/share/vm/opto/graphKit.cpp)中.

```c
//------------------------------builtin_throw----------------------------------
void GraphKit::builtin_throw(Deoptimization::DeoptReason reason, Node* arg) {
  bool must_throw = true;

  if (JvmtiExport::can_post_exceptions()) {
    // Do not try anything fancy if we're notifying the VM on every throw.
    // Cf. case Bytecodes::_athrow in parse2.cpp.
    uncommon_trap(reason, Deoptimization::Action_none,
                  (ciKlass*)NULL, (char*)NULL, must_throw);
    return;
  }

  // If this particular condition has not yet happened at this
  // bytecode, then use the uncommon trap mechanism, and allow for
  // a future recompilation if several traps occur here.
  // If the throw is hot, try to use a more complicated inline mechanism
  // which keeps execution inside the compiled code.
  bool treat_throw_as_hot = false;
  ciMethodData* md = method()->method_data();

  if (ProfileTraps) {
    if (too_many_traps(reason)) {
      treat_throw_as_hot = true;
    }
    // (If there is no MDO at all, assume it is early in
    // execution, and that any deopts are part of the
    // startup transient, and don't need to be remembered.)

    // Also, if there is a local exception handler, treat all throws
    // as hot if there has been at least one in this method.
    if (C->trap_count(reason) != 0
        && method()->method_data()->trap_count(reason) != 0
        && has_ex_handler()) {
        treat_throw_as_hot = true;
    }
  }

  // If this throw happens frequently, an uncommon trap might cause
  // a performance pothole.  If there is a local exception handler,
  // and if this particular bytecode appears to be deoptimizing often,
  // let us handle the throw inline, with a preconstructed instance.
  // Note:   If the deopt count has blown up, the uncommon trap
  // runtime is going to flush this nmethod, not matter what.
  if (treat_throw_as_hot
      && (!StackTraceInThrowable || OmitStackTraceInFastThrow)) {
    // If the throw is local, we use a pre-existing instance and
    // punt on the backtrace.  This would lead to a missing backtrace
    // (a repeat of 4292742) if the backtrace object is ever asked
    // for its backtrace.
    // Fixing this remaining case of 4292742 requires some flavor of
    // escape analysis.  Leave that for the future.
    ciInstance* ex_obj = NULL;
    switch (reason) {
    case Deoptimization::Reason_null_check:
      ex_obj = env()->NullPointerException_instance();
      break;
    case Deoptimization::Reason_div0_check:
      ex_obj = env()->ArithmeticException_instance();
      break;
    case Deoptimization::Reason_range_check:
      ex_obj = env()->ArrayIndexOutOfBoundsException_instance();
      break;
    case Deoptimization::Reason_class_check:
      if (java_bc() == Bytecodes::_aastore) {
        ex_obj = env()->ArrayStoreException_instance();
      } else {
        ex_obj = env()->ClassCastException_instance();
      }
      break;
    }
    if (failing()) { stop(); return; }  // exception allocation might fail
    if (ex_obj != NULL) {
      // Cheat with a preallocated exception object.
      if (C->log() != NULL)
        C->log()->elem("hot_throw preallocated='1' reason='%s'",
                       Deoptimization::trap_reason_name(reason));
      const TypeInstPtr* ex_con  = TypeInstPtr::make(ex_obj);
      Node*              ex_node = _gvn.transform(new (C, 1) ConPNode(ex_con));

      // Clear the detail message of the preallocated exception object.
      // Weblogic sometimes mutates the detail message of exceptions
      // using reflection.
      int offset = java_lang_Throwable::get_detailMessage_offset();
      const TypePtr* adr_typ = ex_con->add_offset(offset);

      Node *adr = basic_plus_adr(ex_node, ex_node, offset);
      Node *store = store_oop_to_object(control(), ex_node, adr, adr_typ, null(), ex_con, T_OBJECT);

      add_exception_state(make_exception_state(ex_node));
      return;
    }
  }
```

根据上面这段源码的switch-case部分可知，JVM只对几个特定类型异常开启了Fast Throw优化，这些异常包括：

- NullPointerException
- ArithmeticException
- ArrayIndexOutOfBoundsException
- ArrayStoreException
- ClassCastException

## demo

NullPointerException的堆栈在没有设置`OmitStackTraceInFastThrow`JVM选项的情况下,会在执行代码一段时间后消失。通过添加JVM选项，堆栈永远不会消失。

```java
public class TestOmitStackTraceInFastThrow {
    public static void main(String[] args) {
        int counter = 0;
        int c=0;
        while(true) {
            try {
                Object obj = null;
                /*
                 * If we cause the exception every time(= without this "if" statement),
                 * the optimization does not happen somehow.
                 * So, cause it conditionally.
                 */
                if(counter % 2 == 0) {
                    obj = new Object();
                }
                // Cause NullpointerException
                obj.getClass();
            }catch(NullPointerException e) {
                e.printStackTrace();
                if(e.getStackTrace() == null || e.getStackTrace().length==0 ){
                    c++;
                    if(c>2){
                        break;
                    }
                }
            }
            counter++;
        }
    }
}
```

