# 反射修改 static final 字段

由于某些需求，在单元测试中需要对 `static final Logger logger` 这个字段/对象 进行强行修改，使用自定义的对象以直接开启 `debugEnabled` 提升测试的代码覆盖率。

那么我们原本经常使用反射来修改字段，即使是 `private` 的访问权限也可以使用 `setAccessible(true);` 轻松绕过限制，直接进行粗暴的修改。甚至，使用 `final` 修饰的非编译前常量字段，也可以被直接修改。但是同时被 `static` 和 `final` 修饰的字段，则会被 `UnsafeQualifiedStaticFieldAccessorImpl` 被设为只读，代码如下，其中 var2 就是 isFinal  &&  (isStatic || !isOverride) 。感觉大部分时候咱们都是 Override 哈。

```java
abstract class UnsafeQualifiedStaticFieldAccessorImpl extends UnsafeStaticFieldAccessorImpl {
    protected final boolean isReadOnly;

    UnsafeQualifiedStaticFieldAccessorImpl(Field var1, boolean var2) {
        super(var1);
        this.isReadOnly = var2;
    }
}
```

所以我们需要对同时被 `static` 和 `final` 修饰的字段做一些额外的骚操作。那就是通过反射获取描述符，并且将

`final` 的标识删除掉，代码如下。

```java
		/**
     * 强制将 target 中被 static final 修饰的 logger 字段修改为可控的实现
     * 目的： 可遍历 logger.debug 方法
     * @param target 测试对象
     * @param myLogger 自实现的logger
     * @throws NoSuchFieldException
     * @throws IllegalAccessException
     */
    private void forceChangeTargetLogger(Object target, Logger myLogger) throws NoSuchFieldException, IllegalAccessException {
        Class<?> targetClazz=target.getClass();
        Field field=targetClazz.getDeclaredField("logger");
        field.setAccessible(true);
        // 获取描述符
        Field modifiers = field.getClass().getDeclaredField("modifiers");
        modifiers.setAccessible(true);
        // 删除描述符中的 final 标识
        modifiers.setInt(field, field.getModifiers() & ~Modifier.FINAL);
        // 修改 static 对象 「 /滑稽 」
        field.set(target, myLogger);
        // 恢复 final 描述符
        modifiers.setInt(field, field.getModifiers() ^ Modifier.FINAL);
        modifiers.setAccessible(false);
        field.setAccessible(false);
				// 但是即使恢复了 final 后面再次修改此字段时也不会抛出异常，可以随意修改！
    }
```

如注释所描述，修改完以后当然就生效了。

而我想通过再次修改描述符的方式使得以后应该不能直接修改这个字段，但事实证明，即使被修改过的字段仍被 `static` 和 `final` 修饰，反射也可以直接调用  field.set(target, myLogger); 不需要上面繁琐的操作了！

经过调试发现，是否拒绝对字段的修改，最最直接的是 `UnsafeStaticObjectFieldAccessorImpl` 对象决定的，这个对象在构造时确实会检查将被修改的字段是否被 `static` 和 `final` 修饰，而一旦经过初始化后，这个对象会被缓存下来，下次轮到这个字段的修改时，仍然是 `UnsafeStaticObjectFieldAccessorImpl` 对象来判断访问权限，而这个对象的 `protected final boolean isFinal` 值为 false，这不是因为我们之前为了修改字段而做的骚操作嘛！所以是清理不够干净。

那至于怎么销毁和释放此 `UnsafeStaticObjectFieldAccessorImpl` 对象，以后有需求了在研究吧！