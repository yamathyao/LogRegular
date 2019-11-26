# LogRegular
Some log regular
# 异常处理

不要捕获Java类库中定义的继承自 RuntimeException 的运行时异常类。
如：IndexOutOfBoundsException/NullPointerException，这类异常由开发人员预检查来规避，保证程序的健壮性。

```
// 正例
if (obj != null) { ... }
 
// 反例
try { obj.method() } catch (NullPointerException e) { ... }
```

*【强制】* 异常不要用来做流程控制，条件控制。因为异常处理效率比条件分支低。

*【强制】* 一大段代码 try-catch 是不负责的表现，catch 时要分清稳定代码合非稳定代码。稳定代码指的是无论如何不会出错的代码。对于非稳定的 try-catch 尽可能进行区分异常类型，再做对应的异常处理。
*【强制】* 捕获异常是为了处理它，不要捕获了却什么都不处理而抛弃之。如果不想处理，请将该异常抛给它的调用者。最外层的业务使用者，必须处理异常，将其转化为可以理解的内容。
 有 try 放到了事务代码中，catch 异常后，如果要回滚事务，一定要注意手动回滚事务。
*【强制】* finally 块必须对资源对象、流对象进行关闭，有异常也要做 try-catch 。
说明：如果JDK7，可以使用 try-with-resources 方法
```
// 传统方式
public static void main(String[] args) {
    FileInputStream fis = null;
    try {
        fis = new FileInputStream(new File("test"));
        System.out.println(fis.read());
    } catch (IOException e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        if (fis != null) {
            try {
                fis.close();
            } catch (IOException e) {
                throw new RuntimeException(e.getMessage(), e);
            }
        }
    }
}
 
// JDK7及之后
public static void main(String[] args) {
    try (FileInputStream fis = new FileInputStream(new File("test"))) {
        System.out.println(fis.read());
    } catch (IOException e) {
        throw new RuntimeException(e.getMessage(), e);
    }
}
```

*【强制】* 不能在 finally 块中使用 return， finally 块中的 return 返回后方法结束执行，不会再执行 try 块中的 return 语句。
*【强制】* 捕获异常与抛异常，必须完全匹配。捕获异常必须是抛异常的父类。（如果抛的是绣球，接的是铅球，就会发生意外）
*【推荐】* 方法的返回值可以为 null，不强制返回空集合，或空对象等，必须添加注释充分说明什么情况下会返回 null 值。调用方需要进行 null 值判断，防止 NPE（NullPointerException） 问题。
防止 NPE 是调用者的责任。即使被调用方法返回空集合或空对象，对调用者来说，也并非高枕无忧，必须考虑到远程调用失败，运行时异常等情景返回 null 的情况。
防止 NPE 是程序员的基本修养，注意 NPE 产生的场景。

a. 返回类型为包装数据类型，有可能是 null ，返回 int 时注意判空。

```
// 反例
public int f() { return Integer对象; }
```
b. 数据库查询结果可能为 null
c. 集合里的元素即使 isNotEmpty，取出的元素也可能是 null
d. 远程调用返回对象，一律要求进行 NPE 判断。
e. 对于 session 中获取的数据，建议 NPE 检查，避免空指针。
f. 级联调用 obj.getA().getB().getC(); 一连串调用，易产生 NPE。

*【推荐】* 在代码中使用“抛异常”还是“返回错误码”，对于公司外的 http/api 开放jiek必须使用“错误码”；而应用内部推荐异常抛出。跨应用间服务调用优先考虑使用Result方式，封装isSuccess、“错误码”、“错误简短信息”。
关于服务方法返回方式使用 Result 方式的理由：
a. 中间件平台基本上使用 ResultDO 来封装，由于中间件的普及，本身就有标准的引导含义。
b. 使用抛异常返回方式，调用方如果没有捕获到就会产生运行时错误。
*【推荐】* 定义时区分 unchecked/checked 异常，避免直接使用 RuntimeException 抛出，更不允许抛出 Exception 或者 Throwable，应使用有业务含义的自定义异常。业界定义过的自定义异常如 DaoException / ServiceException。
*【参考】* 避免出现重复代码（DRY原则 Dont Repeat Yourself）。随意复制和粘贴代码，必然会导致代码的重复，在以后需要修改时，需要修改所有的副本，容易遗漏。必要时抽取共性方法，或者抽象公共类，甚至是公共模块。

# 日志规约

*【强制】* 应用中不可直接使用日志系统（Log4j， LogBack）中的API，而应依赖使用日志框架（SLF4J）中的API。使用门面模式的日志框架，有利于维护和各个类的日志处理方式统一。

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
private static final Logger logger = LoggerFactory.getLogger(Abc.class);

*【强制】* 对 trace/debug/info 级别的日志输出，必须使用条件输出形式或者使用占位符的方式。说明： logger.debug("Processing trade with id: " + id + " symbol: " + symbol); 如果日志级别是warn，上述日志不会打印，但是会执行字符串操作，如果 symbol 是对象，会执行 toString() 方法，浪费了系统资源，执行了上述操作，最终日志却没有打印。

```
// 正例 （条件）
if (logger.isDebugEnabled()) { logger.debug("Processing trade with id: " + id + " symbol: " + symbol); }

// 正例 （占位符）
logger.debug("Processing trade with id: {} and symbol: {}", id, symbol);
```

输出的 POJO 类必须重写 toString 方法，否则输出此对象的 hashCode 值没什么参考意义。
*【推荐】* 可以使用 warn 日志级别来记录用户输入参数错误的情况。注意日志输出的级别， error 级别只记录系统逻辑出错、异常、或者重要的错误信息。如非必要，请不要在此场景打出 error 级别，避免频繁报警。
*【推荐】* 谨慎地记录日志。生产环境禁止输出 debug 日志；有选择地输出 info 日志；如果使用 warn 来记录刚上线时的业务行为信息，一定要注意日志输出量的问题。
说明： 大量输出无效日志，不利于系统性能提升，也不利于快速定位错误点。记录日志时请思考，这些日志真的有人看吗？看到这条日志能干什么？能不能给问题排查带来好处？
