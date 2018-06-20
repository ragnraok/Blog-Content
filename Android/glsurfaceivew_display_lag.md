Title: 浅谈Android中Surface显示延迟处理
Date: 2018-6-20
Slug: surface_display_lag

最近在使用GLES在Surface上渲染结果的时候，遇到了一个显示延迟的问题，当渲染时间过长，同时帧率要求比较高的时候，如果不加控制，就会造成显示的延迟，如果界面没有交互，倒是不会看出什么问题，如果是强交互的场景，这种情况下就会造成可感知的用户使用延迟了。处理这种情况的其中一种方式是给渲染的一帧赋予一个时间戳，用于给SurfaceFlinger进行对应的丢帧控制，具体什么原理呢，让我们一步一步来说明一下

### Surface是什么？

---

从原来上讲，Android的 [Surface](https://developer.android.com/reference/android/view/Surface) 可以认为是目前图形架构中，作为 [BufferQueue](https://source.android.com/devices/graphics/arch-bq-gralloc) 的生产者端，Android会首先把内容渲染到Surface上，填充数据到GraphicBuffer上。而作为消费者端则为系统的[SurfaceFlinger](https://source.android.com/devices/graphics/arch-sf-hwc) ，取出BufferQueue中的GraphicBuffer，并配合vysnc将数据送给HWC合成到屏幕上。

一般来说，Android会把所有可见的View渲染到一个由SurfaceFlinger创建的Surface上，但是这个Surface并不能由开发者直接操作，从App的开发角度来看，大部分情况下我们直接操作的Surface一般会从以下两个地方获取

1. [SurfaceView](https://developer.android.com/reference/android/view/SurfaceView) / [GLSurfaceView](https://kapeli.com/dash_share?docset_file=Android&docset_name=Android&path=developer.android.com/reference/android/opengl/GLSurfaceView.html&platform=android&repo=Main&source=https://developer.android.com/reference/android/opengl/GLSurfaceView.html&version=8.1.0)

   这两个组件是结合了Surface跟View的实现，特别的是，这两个view系统为其单独提供一层了Surface，并直接由SurfaceFlinger进行管理合成，因此实际显示在屏幕上的时候，并没有完全从属在当前的View的布局层次上，在布局对应的位置上，只是一个透明的占位符。而GLSurfaceView，则在SurfaceView的基础上提供了EGL的上下文，以便可以直接使用GLES在Surface上绘制内容

2. [SurfaceTexture](https://developer.android.com/reference/android/graphics/SurfaceTexture) / [TextureView](https://developer.android.com/reference/android/view/TextureView)

   SurfaceTexture是从Android 3.0+开始提供的组件，提供了一个Surface跟GLES纹理的组合，而TextureView，则是一个SurfaceTexture跟View结合起来的组件。而TextureView跟SurfaceView最大的不同在于，虽然都是可以作为BufferQueue的生产方，但是最后合成的时候，并非由SurfaceFlinger直接合成，而是通过GLES直接合成到App对应的Surface上，在布局层次上是跟当前App的View是在同一个层级，对应的View的刷新逻辑也会影响TextureView。因此，从原理上来说，SurfaceView/GLSurfaceView的渲染效率要比TextureView要高

### 使用GLES在Surface中显示内容

---

上面说到，Surface作为BufferQueue的生产方，开发者可以在上面绘制画面，而在SurfaceView/TextureView上，系统都提供了对应的``lockCanvas``方法，返回一个``Canvas``对象允许在对应的Surface上绘制内容。除了这个方法以外，我们也可以使用GLES在Surface上绘制内容。

无论是使用GLSurfaceView，还是TextureView，使用GLES在Surface上绘制内容，我们都必须在单独线程进行GLES的上下文初始化（因为GL Context是跟线程绑定的），创建对应的[EGLSurface](https://source.android.com/devices/graphics/arch-egl-opengl) ，一般来说，实现的代码如下（如果用的是GLSurfaceView，下面的初始化逻辑内部已经给你做好了）：

```kotlin
mEGLDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY)
if (mEGLDisplay === EGL14.EGL_NO_DISPLAY) {
        throw RuntimeException("unable to get EGL14 display")
}
val version = IntArray(2)
if (!EGL14.eglInitialize(mEGLDisplay, version, 0, version, 1)) {
        throw RuntimeException("unable to initialize EGL14")
}

// Configure EGL for recording and OpenGL ES 2.0.
val attribList = intArrayOf(EGL14.EGL_RED_SIZE, 8, 
                            EGL14.EGL_GREEN_SIZE, 8, 
                            EGL14.EGL_BLUE_SIZE, 8, 
                            EGL14.EGL_ALPHA_SIZE, 8,
                            EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT,                 
                            EGL_RECORDABLE_ANDROID, 1, 
                            EGL14.EGL_NONE)
val configs = arrayOfNulls<EGLConfig>(1)
val numConfigs = IntArray(1)
EGL14.eglChooseConfig(mEGLDisplay, attribList, 0, configs, 0, configs.size, numConfigs, 0)
checkEglError("eglCreateContext RGB888+recordable ES2")

// Configure context for OpenGL ES 2.0.
val attrib_list = intArrayOf(EGL14.EGL_CONTEXT_CLIENT_VERSION, 2, EGL14.EGL_NONE)
mEGLContext = EGL14.eglCreateContext(mEGLDisplay, configs[0], EGL14.EGL_NO_CONTEXT, attrib_list, 0)
checkEglError("eglCreateContext")

// Create a window surface, and attach it to the Surface we received.
val surfaceAttribs = intArrayOf(EGL14.EGL_NONE)
mEGLSurface = EGL14.eglCreateWindowSurface(mEGLDisplay, configs[0], mSurface, surfaceAttribs, 0)
checkEglError("eglCreateWindowSurface")

EGL14.eglMakeCurrent(mEGLDisplay, mEGLSurface, mEGLSurface, mEGLContext)
checkEglError("eglMakeCurrent")
```

以上代码摘自[这里](https://bigflake.com/mediacodec/CameraToMpegTest.java.txt)，并转换成了kotlin

大概流程就是先选择好需要的EGL配置，然后初始化EGLConfig跟EGLContext，最终，调用``eglCreatexxxSurface``创建一个EGLSurface，这这个例子中调用的是``eglCreateWindowSurface``，并在函数中传入了SurfaceView/TextureView中的Surface对象

另外，在`eglCreateWindowSurface`函数中传入的Surface对象，如果不是需要渲染到屏幕上的话，除了直接使用上面的两个Surface对象以外，在很多处理视频特效的应用中，另外一种方式是传入MediaCodec的Surface，也就是这个方法的返回结果

```java
MediaCodec.getInputSurface
```

然后MediaCodec作为编码器使用，渲染一帧之后结果就直接编码到结果视频中了

- *这里简单在说明一点，在这种场景下（还有一种是使用GLES渲染camera preview），使用GLES渲染对应MediaCodec编码结果到Input Surface上的时候，使用的纹理类型必须是外部纹理（``GL_TEXTURE_EXTERNAL_OES``）*

在上面的初始化代码中，最终结果是创建了一个``EGLSurface``对象，这个对象最终会链接到Surface中的BufferQueue生产方接口，渲染到该EGLSurface上的新的一帧将会让一个GraphicBuffer离开队列并提供给消费者一方使用，但是，EGL并不会自动给提交当前渲染的一帧，当渲染好之后，需要调用``eglSwapBuffers``提交当前渲染结果，从而实现BufferQueue的刷新

结合SurfaceFlinger，使用GLES在Surface上渲染，上屏的整理流程大概如下：

**Surface -> EGL renderer -> swap buffer -> BufferQueue deque -> SurfaceFlinger -> HWC -> Display**

更具体的GraphicBuffer/BufferQueue同步机制，推荐看下[这篇](https://blog.csdn.net/jinzhuojun/article/details/39698317)文章的分析

### Surface渲染上屏时间戳

---

上面大体分析了Surface原理，以及对应渲染上屏的步骤，但是，直到SurfaceFlinger，到Hareware Composer这一步，我们还有一个关键的问题没有解决：

**我们在渲染好一帧之后，如何能够保证这一帧的内容能够及时显示到屏幕上呢？**

或者换一个问法：

**当我们渲染一帧的时间过长的时候，我们又怎么能够保证在对应的时间点上在屏幕上显示对应的内容呢？**

如果没有解决这个问题，那么在游戏渲染，或者在视频播放渲染的时候，就很容易出现音视频不同步的情况。而Android对于这个问题的解决方式就是，让App去告诉SurfaceFlinger某一帧想要在哪个时间点显示到屏幕上，也就是说引入了帧时间戳的概念。当SurfaceFlinger提交到HWC超过这个时间戳的时候，就丢掉这一帧，如果还没达到对应帧的时间戳，就继续显示当前帧

而在实现上，Android提供了一个单独的EGL扩展：[eglPresentationTimeANDROID](https://www.khronos.org/registry/EGL/extensions/ANDROID/EGL_ANDROID_presentation_time.txt) ，在swapBuffer之前调用，提交当前帧的想要的显示时间戳，至于时间戳的具体含义，在不同的场景中可能会有不同的表达，例如：

1. 如果是显示到屏幕上的时候，时间戳就是一个绝对时间值，例如系统的启动时间

2. 如果是视频编码的场景，例如使用MediaCodec的InputSurface来编码视频的时候，这个时候时间戳的含义就是当前视频帧的 [pts](https://en.wikipedia.org/wiki/Presentation_timestamp)，事实上，当你想在MediaCodec的InputSurface上渲染完内容之后，如果不调用这个函数控制当前这一帧的pts，除非合成器有额外控制，否则最后编码出来的视频fps将会相当大，具体这里的实现，可以参考下BigFlake这里的[代码](https://bigflake.com/mediacodec/EncodeAndMuxTest.java.txt)

*btw, 这个扩展对应的Android上层接口定义在[这里](https://developer.android.com/reference/android/opengl/EGLExt.html)*

因此，通过对 ``eglPresentationTimeANDROID`` 的调用，结合BufferQueue，SurfaceFlinger就可以针对上屏的每一帧数据延迟做精确的控制了，假设说，我们设置了某一帧显示时间戳为**T**，然后提交到BufferQueue中：

1. 当在**T-1**的时间点，当前队首为这一帧的时候，SurfaceFlinger会继续hold住当前帧，也就是说这个时候显示的还是前一帧的数据

2. 当达到了**T**时间点，当前队首为这一帧的时候，SurfaceFlinger便直接提交这一帧到Display

3. 当达到了**T+1**时间点，当前队首为这一帧的时候，因为已经超过了这一帧设置的时间戳**T**，因此SurfaceFlinger便直接**丢弃**这一帧，继续处理队列剩余的帧数据

总体来看，在通过帧时间戳控制之后，Android就可以解决Surface的渲染上屏延迟问题，但渲染过长的时候，就势必带来丢帧，因此根本的解决方案，还是得尽量在16ms内，渲染完一帧数据

文笔一般，水平有限，仅做抛砖引玉之用，欢迎更加仔细的讨论!

### References

---

- [StackOverFlow—Minimize Android GLSurfaceView Lag](https://stackoverflow.com/questions/26317132/minimize-android-glsurfaceview-lag/)

- [Android中的GraphicBuffer同步机制-Fence](https://blog.csdn.net/jinzhuojun/article/details/39698317)

- [EGL_ANDROID_presentation_time](https://www.khronos.org/registry/EGL/extensions/ANDROID/EGL_ANDROID_presentation_time.txt)

- [Android图形架构](https://source.android.com/devices/graphics/)

- [Android MediaCodec Stuff](https://bigflake.com/mediacodec/)
