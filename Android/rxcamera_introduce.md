Title: RxCamera,  一个RxJava风格的android camera封装
Date: 2015-12-20
Slug: rxcamera-introduce

事实上这个库写了已经有一段时间了，由于最近工作上比较忙，所以现在才写一篇文章来总结

###What's this
正如标题所说，[RxCamera](https://github.com/ragnraok/RxCamera)是一个基于RxJava而构建的一套[android.hardware.camera](http://developer.android.com/intl/es/reference/android/hardware/Camera.html)封装的库。最初写这个库的目的是为了熟悉RxJava，并且而且也看到虽然在android开发已经有不少基于RxJava的库，但是关于音视频相关却少之又少，于是就动手实现了一下。目前这个库还处于非常早期的状态，API比较简陋，并且关于camera的设置还有很多没有做

###How to use
在[RxCamera](https://github.com/ragnraok/RxCamera)的README中已经有关于这个库使用的比较详细介绍了，我在这里再说明一下：

####加入依赖
首先你需要在项目中加入对RxCamera项目的依赖:

```
repositories {
        jcenter()
}
dependencies {
    compile 'com.ragnarok.rxcamera:lib:0.0.1'
}
```

####基本的使用

- 配置camera的参数

android的原生camera api提供了不少的选项来配置打开摄像头时候的参数，例如预览的帧率，预览的分辨率，自动对焦等，在RxCamera中主要是通过一个``RxCameraConfig``对象来管理这些对象，并通过``RxCameraConfigChooser``来配置对应参数：
	
```
	RxCameraConfig config = RxCameraConfigChooser.obtain().
        useBackCamera().
        setAutoFocus(true).
        setPreferPreviewFrameRate(15, 30).
        setPreferPreviewSize(new Point(640, 480)).
        setHandleSurfaceEvent(true).
        get();
```
	
-  打开摄像头并获取数据

在设置好参数之后，就可以直接打开摄像头了：
	
```
	RxCamera.open(this, config).flatMap(new Func1<RxCamera, Observable<RxCamera>>() {
      @Override
      public Observable<RxCamera> call(RxCamera rxCamera) {
          return rxCamera.bindTexture(textureView);
          // or bind a SurfaceView:
          // rxCamera.bindSurface(SurfaceView)
      }
	 }).flatMap(new Func1<RxCamera, Observable<RxCamera>>() {
        @Override
         public Observable<RxCamera> call(RxCamera rxCamera) {
            return rxCamera.startPreview();
         }
    });
```
	
这里包括了设置用于预览的Surface(这里是使用``TextureView``进行预览)，然后正式开始预览。
	
只有预览之后才能开始获取摄像头的数据：
	
```
	camera.request().periodicDataRequest(1000).subscribe(new Action1<RxCameraData>() {
            @Override
            public void call(RxCameraData rxCameraData) {
                showLog("periodic request, cameraData.length: " + rxCameraData.cameraData.length);
            }
    });
```
	
获取摄像头的数据都通过``request``来获取， RxCamera中封装了几种不同风格的cameraRequest，例如上面的是定时获取摄像头数据，每隔1000毫秒返回一次，另外还有连续返回数据，只返回一次数据等的request。这些request返回的Observable对象都会给订阅者返回``RxCameraData``对象，其中包含两个字段
	
1.  `` byte[] cameraData``，就是原生的摄像头数据，具体的数据格式，如果是从预览数据中获取，则是设置的previewFormat格式，没有设置则默认为YUV12。如果是takePictureRequest的话，则是返回JPEG格式的数据
	
2.  ``Matrix rotateMatrix``，这个Matrix可以帮助你把返回的摄像头数据旋转回在竖屏模式下正常显示，因为大部分android手机的安装角度都会90度或者270度，这样子在横屏拍摄的时候可以拿到正确的图像，但是在竖屏显示的时候就会被旋转过来，使用这个Matrix旋转获取到的摄像头数据之后，就可以获取到正确转向的图像了

###目前的状态

很明显这个库目前还处于非常早期的状态，API还比较简单，并且很多摄像头的参数也没有加入到进行设置，例如白平衡，闪光灯，测光等，后续这里将会逐渐完善丰富摄像头的各种设置。另外，由于是完全基于RxJava来构建的库，大部分接口都会直接返回Observable对象，这对于没有熟悉RxJava的人来说可能还是会有点距离，因此这里后续对接口的封装还是需要继续改进，对外可能不再暴露Observable对象

###About camera2

熟悉android开发的人估计都已经知道在Lollipop，Google新增一套[camera2](http://developer.android.com/intl/es/reference/android/hardware/camera2/package-summary.html)API用以取代之前的camera API，camera2提供了对摄像头更加精细化的控制，相对来说比起老的API，对摄像头的可控程度高了非常多。但我个人觉得新的API还不是特别好用，并且由于目前还是比较新，在使用新的camera2 API的时候，[还需要查询手机上有哪些特性是有实现的](http://source.android.com/devices/camera/versioning.html#camera_api2_capabilities_and_support_levels)，目前使用起来相当麻烦。所以RxCamera暂时还不打算支持camera2

