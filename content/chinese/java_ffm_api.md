+++
title = "浅析一下Java FFM API(Project Panama)"
date = "2023-08-05"
taxonomies.tags = [
    "JVM",
    "Chinese",
]
+++

**这篇文章并不是讲如何使用Java FFM API，而是浅谈其背后的实现原理。**

## 前言

前不久，OpenJDK宣布了```Java Foreign Function & Memory API```将在JDK 22退出预览，这意味着在JDK 22后，FFM API不会有重大改动。借此机会，我想可以好好聊聊FFM API是怎么实现的。

## FFM API介绍

FFM API由两大部分组成，一个是```Foreign Function Interface```，另一个是```Memory API```。前者是外部函数接口，简称FFI，用它来实现Java代码和外部代码之间相互操作；后者是内存接口，用于**安全地**管理堆外内存。

## Memory API

为了方便切入，我这里写了一个很简单的Demo：
```java
private static void allocDemo() throws Throwable {
    try (Arena arena = Arena.ofConfined()) {
        MemorySegment cString = arena.allocateUtf8String("Panama");
        String jString = cString.getUtf8String(0l);
        System.out.println(jString);
    }
}
```
- 这里在堆外开辟了一段内存来存放字符串```Panama```，接下来copy到JVM栈上，最后输出。
- 简单介绍下，```MemorySegment```用于表示一段内存片段（既可以是堆内也可以是堆外），```Arena```划定了一个作用域，便于进行内存回收。

这背后做了些什么工作呢？我们跳过去看看。

### Memory API是对jdk.internal.misc.Unsafe的安全封装

我们首先跳到```cString.getUtfString```，因为很明显该部分涉及到访问堆外内存的操作。局部代码如下：

```MemorySegment.java line:1089```

```java
default String getUtf8String(long offset) {
    return SharedUtils.toJavaStringInternal(this, offset);
}
```

- 这里写把操作写到了一个Util里面。

我们跳过去看看。

---

```ShareUtils.java line:250```

```java
public static String toJavaStringInternal(MemorySegment segment, long start) {
    int len = strlen(segment, start);
    byte[] bytes = new byte[len];
    MemorySegment.copy(segment, JAVA_BYTE, start, bytes, 0, len);
    return new String(bytes, StandardCharsets.UTF_8);
}
```

- 这里设置了一个```byte```数组，这是在JVM栈上的，之后访问堆外内存，进行copy操作。

很明显```MemorySegment.copy```才是我们关心的，再跳过去看看。

---

```MemorySegment.java line:2209```
```java
@ForceInline
static void copy(
        MemorySegment srcSegment, ValueLayout srcLayout, long srcOffset,
        Object dstArray, int dstIndex, int elementCount) {
    Objects.requireNonNull(srcSegment);
    Objects.requireNonNull(dstArray);
    Objects.requireNonNull(srcLayout);

    AbstractMemorySegmentImpl.copy(srcSegment, srcLayout, srcOffset,
            dstArray, dstIndex,
            elementCount);
}
```

- 这里先是一顿判空，之后再调用了```AbstractMemorySegmentImpl.copy```方法，这里```AbstractMemorySegmentImpl```是```MemorySegment```的实现，```MemorySegment```是一个密封接口，只允许了```AbstractMemorySegmentImpl```实现。

OK，继续跳转。

---

```AbstractMemorySegmentImpl.java line:625```
```java
@ForceInline
public static void copy(MemorySegment srcSegment, ValueLayout srcLayout, long srcOffset,
                        Object dstArray, int dstIndex,
                        int elementCount) {
// 此处省略了原本一系列判空、校验等操作。
    if (dstWidth == 1 || srcLayout.order() == ByteOrder.nativeOrder()) {
        ScopedMemoryAccess.getScopedMemoryAccess().copyMemory(srcImpl.sessionImpl(), null,
                srcImpl.unsafeGetBase(), srcImpl.unsafeGetOffset() + srcOffset,
                dstArray, dstBase + (dstIndex * dstWidth), elementCount * dstWidth);
    } else {
        ScopedMemoryAccess.getScopedMemoryAccess().copySwapMemory(srcImpl.sessionImpl(), null,
                srcImpl.unsafeGetBase(), srcImpl.unsafeGetOffset() + srcOffset,
                dstArray, dstBase + (dstIndex * dstWidth), elementCount * dstWidth, dstWidth);
    }
}
```

- 这里可以看到一个```ScopedMemoryAccess```，熟悉jdk.internal.misc.Unsafe的大佬估计到这就懂了，```ScopedMemoryAccess```也是在jdk.internal.misc下的，并且其中有一个```UNSAFE```字段。

最终，我们顺着```ScopedMemoryAccess.getScopedMemoryAccess().copyMemory```最终会跳转到哪里呢？答案是```Unsafe```中的一个native方法：```copyMemory0```，~~而该native方法是通过JNI实现的~~虽然它是一个native方法，但是它被`@IntrinsicCandidate`注解修饰。

#### @IntrinsicCandidate
这个注解表明被修饰的方法**可能**会被HotSpot内联，这取决于是否为当前平台的虚拟机手写了汇编/编译器IR，以降低开销。

部分jdk.internal.misc.Unsafe的native方法如下：

`Unsafe.java line:3823`
```java
private native long allocateMemory0(long bytes);
private native long reallocateMemory0(long address, long bytes);
private native void freeMemory0(long address);
private native void setMemory0(Object o, long offset, long bytes, byte value);
@IntrinsicCandidate
private native void copyMemory0(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes);
private native void copySwapMemory0(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes, long elemSize);
private native long objectFieldOffset0(Field f);
```
- 可以看到像`Unsafe::allocateMemory0`, `Unsafe::freeMemory0`等关键的native方法未被`@IntrinsicCandidate`注解修饰，这也意味着这些方法是使用JNI实现的。

那么Memory API是否使用了这些native方法呢？

经过分析，Memory API开辟堆外内存的操作确实使用了`Unsafe::allocateMemory0`方法。

OK，可以下结论了：Memory API是对```jdk.internal.misc.Unsafe```的封装，使得Java程序员操纵堆外内存更加得心应手，让Unsafe变得Safe。这一点其实已在相关的JEP里表明了：

> ## Non-goals
>
> It is not a goal to
>
> · Re-implement JNI on top of this API, or otherwise change JNI in any way;
> 
> · Re-implement legacy Java APIs, such as `sun.misc.Unsafe`, on top of this API;
> 
> · Provide tooling that mechanically generates Java code from native-code header files; or
> 
> · Change how Java applications that interact with native libraries are packaged and deployed (e.g., via multi-platform JAR files).

_来源：[JEP 442: Foreign Function & Memory API (Third Preview)](https://openjdk.org/jeps/442)_

我在阅读该段落时陷入过一个逻辑错误，为避免大家同样陷入该逻辑错误，我在这解释一下。

该段第二条讲的是：在该API上，重新实现像```sun.misc.Unsafe```之类的遗留API。而这里是Non-goals，也就是说，该API不会重新实现像```sun.misc.Unsafe```之类的遗留API，意味着该API会做一些像完善```sun.misc.Unsafe```之类的工作。

可以看到，Java FFM API并没有完全脱离JNI。那FFI部分呢？该不会也是封装JNI吧？我们一探究竟。

## Foreign Function Interface

我同样写了一个Demo：

```java
private static void downCallDemo() throws Throwable {
    Linker linker = Linker.nativeLinker();
    MemorySegment strlen_address = linker.defaultLookup().find("strlen").get();
    MethodHandle strlen = linker.downcallHandle(
            strlen_address,
            FunctionDescriptor.of(JAVA_LONG, ADDRESS)
    );
    try (Arena arena = Arena.ofConfined()) {
        MemorySegment cString = arena.allocateUtf8String("Hello");
        long len = (long) strlen.invokeExact(cString); // 5
        System.out.println(len);
    }
}
```

- 这里先是整一个Linker，之后获取本地既有函数```strlen```的地址，接着调用它并传入一个字符串，最后获取它的返回值，输出。

我比较好奇的是，它是如何直接通过一个字符串查找到一个函数的？

### Linker 与 SymbolLookup

为弄清以上问题，我们先看看```linker.defaultLookup()```是怎么实现的，因为很明显这里涉及到查找等操作。

```Linker.java line:636```

```java
SymbolLookup defaultLookup();
```

- 遗憾的是，```Linker```是一个接口，里面并没有实现```defaultLookup()```，因此我们需要找到它的实现类。

在```Linker.java```中可以发现，```Linker```是一个密封接口，仅允许```AbstractLinker```实现。

---

```AbstractLinker.java line:60```

```java
public abstract sealed class AbstractLinker implements Linker permits LinuxAArch64Linker, MacOsAArch64Linker,
                                                                      SysVx64Linker, WindowsAArch64Linker,
                                                                      Windowsx64Linker, LinuxPPC64leLinker,
                                                                      LinuxRISCV64Linker, FallbackLinker {
// ......
}
```

- 可以看到，```AbstractLinker```是个密封类，并提供了多个平台的实现。这里便可以推断出，在某个地方存在根据平台返回不同```Linker```的操作。实际上，这个操作就在```Linker.nativeLinker()``` :(

---

```AbstractLinker```实现了```defaultLookup()```方法：

`AbstractLinker.java line:133`

```java
@Override
public SystemLookup defaultLookup() {
    return SystemLookup.getInstance();
}
```

- 这里是的返回类型是`SystemLookup`，而在`Linker`内，`defaultLookup`的返回类型为`SymbolLookup`。实际上，`SystemLookup`是`SymbolLookup`的实现。

至此，我们可以判断，在我们的Demo中，`linker`为对应平台的`Linker`实现；`linker.defaultLookup()`实际返回一个SystemLookup对象。

那么，在`linker.defaultLookup().find("strlen")`中又发生了什么？

---

`SystemLookup.java line:137`

```java
@Override
public Optional<MemorySegment> find(String name) {
    return SYSTEM_LOOKUP.find(name);
}
```

`SystemLookup.java line:57`

```java
private static final SymbolLookup SYSTEM_LOOKUP = makeSystemLookup();

private static SymbolLookup makeSystemLookup() {
    try {
        if (Utils.IS_WINDOWS) {
            return makeWindowsLookup();
        } else {
            return libLookup(libs -> libs.load(jdkLibraryPath("syslookup")));
        }
    } catch (Throwable ex) {
        // This can happen in the event of a library loading failure - e.g. if one of the libraries the
        // system lookup depends on cannot be loaded for some reason. In such extreme cases, rather than
        // fail, return a dummy lookup.
        return FALLBACK_LOOKUP;
    }
}
```

`SystemLookup.java line:106`

```java
private static SymbolLookup libLookup(Function<RawNativeLibraries, NativeLibrary> loader) {
    NativeLibrary lib = loader.apply(RawNativeLibraries.newInstance(MethodHandles.lookup()));
    return name -> {
        Objects.requireNonNull(name);
        if (Utils.containsNullChars(name)) return Optional.empty();
        try {
            long addr = lib.lookup(name);
            return addr == 0 ?
                    Optional.empty() :
                    Optional.of(MemorySegment.ofAddress(addr));
        } catch (NoSuchMethodException e) {
            return Optional.empty();
        }
    };
}
```

- 不难发现，Demo中调用`linker.defaultLookup().find("strlen")`时，实际返回一个类型为`Optional<MemorySegment>`的对象。
- 在`SystemLookup.libLookup()`中，进行了加载本地库，获取函数地址等操作。我们需要研究的，就是在这个方法内。

### jdk.internal.loader下的本地库相关实例

在以上代码片段中，`NativeLibrary`用于表示已加载的本地库。官方给的解释如下：

> NativeLibrary represents a loaded native library instance.

在以上代码片段中，`RawNativeLibraries`用于管理已加载的本地库。它有一个`libraries`哈希表，显然是用于存放已加载的本地库；提供了`load`，`unload`等方法的实现。

注意，使用`RawNativeLibraries::load`方法加载的本地库，不会被视为JNI本地库，而是被当成一个普通的本地库对待。`RawNativeLIbraries`可以加载JNI本地库，但是JNI本地库中，包含`JNI_OnLoad`和`JNI_OnUnload `函数，这两个函数会被`RawNativeLIbraries`忽略，可能导致无法维持原JNI库的功能，同时不支持将JNI本地库中的函数和native方法链接。官方解释如下：

> RawNativeLibraries has the following properties: 1. Native libraries loaded in this RawNativeLibraries instance are not JNI native libraries. Hence JNI_OnLoad and JNI_OnUnload will be ignored. No support for linking of native method. 2. Native libraries not auto-unloaded. They may be explicitly unloaded via NativeLibraries::unload. 3. No relationship with class loaders

同时，这里存在另外的类用于加载JNI本地库：`NativeLibraries`。

从这我们可以看出来，Project Panama将FFI与JNI做了严格的切割。主要原因有这几点：

1. FFI默认需要加载的本地库不是专门为JVM设计的。
2. FFI要支持本地库可以被不同的Classloader加载。
3. FFI要支持本地库可以根据需要被多次加载。

> 这也是FFI与JNI重要的不同之处，FFI赢太多了。

在以上代码片段中，不难发现，本地函数的地址由`NativeLibrary::lookup()`得到。

实际上，在代码片段`SystemLookup.java line:106`中，`lib`的类型最终为`RawNativeLibraryImpl`。该类为`RawNativeLibraries`的内部类，继承`NativeLibrary`。而`RawNativeLibraries::load`的返回值类型就是`RawNativeLibraryImpl`，因此本地函数的地址由`RawNativeLibraryImpl::lookup()`得到。

但它并没有重写`lookup()`方法，哈哈！而是重写了`find()`方法：

`RawNativeLibraries.java line:168`

```java
@Override
public long find(String name) {
    return findEntry0(handle, name);
}
```

`NativeLibrary.java line:48`

```java
public final long lookup(String name) throws NoSuchMethodException {
    long addr = find(name);
    if (0 == addr) {
        throw new NoSuchMethodException("Cannot find symbol " + name + " in library " + name());
    }
    return addr;
}

/*
    * Returns the address of the named symbol defined in the library of
    * the given handle.  Returns 0 if not found.
    */
static native long findEntry0(long handle, String name);
```

至此，我们已经到达了Java宇宙的边界。显然，这里还是用到了JNI，来查找本地函数的地址。

## 总结

通过这次分析可以看到，FFM API在架构上做出了很大的优化，这一点或许可以说明FFM API的性能优势。

FFI在加载机制上做出了很大改变，大大提高了互操作性。

FFM API并没有独立于jdk.internal.misc.Unsafe和JNI存在，但也不是简单的封装，更不是对JNI的修改，而是一种利用关系。

遗憾的是，这篇文章并没有涉及到`MethodHandle`相关的分析，没有讲Java FFI是如何实现`downcall`和`upcall`的，其中还有很多有趣的技术和解决方案。

> 参考：
>
> [JEP draft: Foreign Function & Memory API (openjdk.org)](https://openjdk.org/jeps/8310626)
>
> [JDK 21 (openjdk.org)](https://openjdk.org/projects/jdk/21/)
