【源码剖析】Android Bitmap 的创建、销毁、内存回收机制
===
# 1. 概览

在Android系统中，Bitmap是一个用于处理图像数据的核心类，它封装了像素数据和相关的元数据（如宽度、高度和颜色格式）。Bitmap的管理直接关系到应用程序的内存使用效率和性能。由于图像数据往往占用较大的内存空间，不当的Bitmap处理极易引发OutOfMemoryError异常。因此，深入理解Bitmap的创建、销毁以及内存回收机制对于开发高性能的Android应用至关重要。

# 2. Bitmap创建

## 2.1 Bitmap的创建API：

- decodeXXX 方法：如BitmapFactory.decodeResource()、BitmapFactory.decodeFile()等，这些方法从不同的资源（如资源文件、文件系统、网络流）中解码图像数据生成Bitmap对象。
- createBitmap 方法：如Bitmap.createBitmap()，可以创建指定大小的空Bitmap或者根据现有Bitmap、数组等创建新的Bitmap实例。
- Bitmap.Config：在创建Bitmap时，通过指定Bitmap.Config（如ARGB_8888、RGB_565等）来决定Bitmap的颜色深度和存储需求，不同的配置直接影响Bitmap的内存占用。

## 2.1 创建过程源码解析


# 3. Bitmap 销毁

## 3.1 Bitmap的销毁API：


Bitmap对象不再使用时，应当及时释放其占用的内存资源，主要通过以下方法：

- recycle()：显式调用Bitmap的recycle()方法，请求系统回收此Bitmap所占用的内存。但需要注意的是，调用后Bitmap对象应被视为无效，不应再被使用。
- 弱引用（WeakReference）：在某些场景下，使用弱引用来持有Bitmap对象，可以让垃圾回收器在内存紧张时自动回收Bitmap。

## 2.1 recycle过程源码解析


# 4. Bitmap内存回收机制剖析

Bitmap初始化的的时候，将native层的析构函数指针、Bitmap对象指针，通过java层的NativeAllocationRegistry进行托管（Java层拿到是地址，long类型的）。
```java
// called from JNI and Bitmap_Delegate. Bitmap 对象是native层new出来的，因为要传入bitmap的内存指针
Bitmap(long nativeBitmap, int width, int height, int density,
        boolean requestPremultiplied, byte[] ninePatchChunk,
        NinePatch.InsetStruct ninePatchInsets, boolean fromMalloc) {
    if (nativeBitmap == 0) {
        throw new RuntimeException("internal error: native bitmap is 0");
    }

    mWidth = width;
    mHeight = height;
    mRequestPremultiplied = requestPremultiplied;

    mNinePatchChunk = ninePatchChunk;
    mNinePatchInsets = ninePatchInsets;
    if (density >= 0) {
        mDensity = density;
    }

    mNativePtr = nativeBitmap;

    final int allocationByteCount = getAllocationByteCount();
    NativeAllocationRegistry registry;
    // 
    if (fromMalloc) {
        registry = NativeAllocationRegistry.createMalloced(
                Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), allocationByteCount);
    } else {
        registry = NativeAllocationRegistry.createNonmalloced(
                Bitmap.class.getClassLoader(), nativeGetNativeFinalizer(), allocationByteCount);
    }
    // 这里有一个返回值没有接收
    registry.registerNativeAllocation(this, nativeBitmap);
    synchronized (Bitmap.class) {
        sAllBitmaps.put(this, null);
    }
}
```
nativeGetNativeFinalizer的netive层JNI实现：将一个delete 指针的函数传递给Java层。
```c++

static void Bitmap_destruct(BitmapWrapper* bitmap) {
    delete bitmap;
}

static jlong Bitmap_getNativeFinalizer(JNIEnv*, jobject) {
    return static_cast<jlong>(reinterpret_cast<uintptr_t>(&Bitmap_destruct));
}
```
registerNativeAllocation 的逻辑：将引用封装到Cheaner中。以下是Cleaner类的注释说明：
> 
>  General-purpose phantom-reference-based cleaners.
> 
>Cleaners are a lightweight and more robust alternative to finalization.
>They are lightweight because they are not created by the VM and thus do not
>require a JNI upcall to be created, and because their cleanup code is
>invoked directly by the reference-handler thread rather than by the
>finalizer thread.  They are more robust because they use phantom references,
>the weakest type of reference object, thereby avoiding the nasty ordering
>problems inherent to finalization.

> Cleaners是一种轻量级且更强大的最终化替代方案。
它们是轻量级的，因为它们不是由VM创建的，因此不是由VM创建的
需要创建JNI upcall，因为它们的清理代码是
由引用处理程序线程直接调用，而不是由
终结器线程。它们更健壮，因为它们使用幻像引用，
引用对象的最弱类型，从而避免令人讨厌的排序
最终确定固有的问题。

>A cleaner tracks a referent object and encapsulates a thunk of arbitrary
>cleanup code.  Some time after the GC detects that a cleaner's referent has
>become phantom-reachable, the reference-handler thread will run the cleaner.
>Cleaners may also be invoked directly; they are thread safe and ensure that
>they run their thunks at most once.

>清理器跟踪引用对象并封装任意的thunk
>清理代码。**在GC检测到清洁工的指涉有一段时间后
>成为虚引用可达，引用处理程序线程将运行清理程序。**
>清理器也可以直接调用；它们是线程安全的，并确保
>他们最多跑一次。
> 
> Cleaners are not a replacement for finalization.  They should be used
>only when the cleanup code is extremely simple and straightforward.
>Nontrivial cleaners are inadvisable since they risk blocking the
>reference-handler thread and delaying further cleanup and finalization.
> 
>@author Mark Reinhold
> 
 
```c++
public @NonNull Runnable registerNativeAllocation(@NonNull Object referent, long nativePtr) {
    if (referent == null) {
        throw new IllegalArgumentException("referent is null");
    }
    if (nativePtr == 0) {
        throw new IllegalArgumentException("nativePtr is null");
    }

    CleanerThunk thunk;
    CleanerRunner result;
    try {
        thunk = new CleanerThunk();
        Cleaner cleaner = Cleaner.create(referent, thunk);
        result = new CleanerRunner(cleaner);
        registerNativeAllocation(this.size);
    } catch (VirtualMachineError vme /* probably OutOfMemoryError */) {
        // 内存溢出 立即释放内存
        applyFreeFunction(freeFunction, nativePtr);
        throw vme;
    } // Other exceptions are impossible.
    // Enable the cleaner only after we can no longer throw anything, including OOME.
    thunk.setNativePtr(nativePtr);
    // Ensure that cleaner doesn't get invoked before we enable it.
    Reference.reachabilityFence(referent); // 这里手动加一个强引用
    return result;
}
```
NativeAllocationRegistry是结合虚拟机的native内存回收模块，内部又一个applyFreeFunction方法，就是让native层拿到要回收的native c++对象的析构函数，然后用c++对象指针执行调用，完成native内存的释放（即使用C++的delete方法）。NativeAllocationRegistry内部有个内部类CleanerThunk集成了基于虚引用的内存回收执行调用的机制。

```c++
static void NativeAllocationRegistry_applyFreeFunction(JNIEnv*,
                                                       jclass,
                                                       jlong freeFunction,
                                                       jlong ptr) {
    void* nativePtr = reinterpret_cast<void*>(static_cast<uintptr_t>(ptr)); // 对象引用
    FreeFunction nativeFreeFunction
        = reinterpret_cast<FreeFunction>(static_cast<uintptr_t>(freeFunction));
    nativeFreeFunction(nativePtr); // 执行析构函数
}
```
当Java层判定Bitmap对象没有强引用的时候，bitmap的占用的内存会随着GC的过程自然被回收。

# 5. 总结


# 6. 参考文献

## 6.1 Android源码

Android is a mobile operating system developed by Google 
https://cs.android.com/android  

## 6.2 前辈们的博客

- [Android Bitmap变迁与原理解析（4.x-8.x）](https://elephanty.top//2018/05/20/Android-Bitmap%E5%8F%98%E8%BF%81%E4%B8%8E%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90-4.x-8.x/)