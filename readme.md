# Android StreamLakeContentCreator接入文档

该demo工程提供了可直接运行的一个app项目，具体的接口调用时序请参照demo工程，包含了生产组件和播放器两个aar。

## 一 集成准备
<b>Step1</b>. AndroidStudio项目配置：  &nbsp;&nbsp;&nbsp;&nbsp;  
需要minSdkVersion>=24

Step2. 混淆配置配置：  
参考demo项目中的app/proguard-rules.pro。

Step3. 导入aar  
导入contentcreator.aar和播放器aar，并依赖这两个aar。具体参见附件里的demo：
:localaar:contentcreator项目里的contentcreator-release.aar是生产组件
:localaar:mediaplayer项目里的streamlake-mediaplayer-1.8.0.9.aar是播放器

Ste4. 依赖公共plugin  
contentcreator依赖以下公共插件
在你的app/build.gradle添加以下依赖
```
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
apply plugin: 'kotlin-kapt'
dependencies {
   implementation 'androidx.appcompat:appcompat:1.1.0'
   kapt "androidx.room:room-compiler:2.2.5"
}
```
如果您的仓库依赖以下组件，请先统一将其做exclude处理
```
configurations.all {
   exclude(group: 'com.facebook.fresco', module: 'imagepipeline-base')
   exclude(group: 'com.google.guava', module: 'listenablefuture')
   exclude group: "com.facebook.fresco", module: "animated-webp"
   exclude group: "com.facebook.soloader", module: "soloader"
   exclude group: "com.facebook.fresco", module: "animated-webp"
}
```


Step5. 添加所需权限权限	  
访问网络  
`<uses-permission android:name="android.permission.INTERNET"/>`

获取网络状态  
`<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>`
   
Step6. 依赖公共三方库  
导入公共依赖，其中版本可能会有冲突，需要针对具体的case做一些class冲突错误的fix，具体参见demo中的 `app/common_dependency.gradle`


至此完成了接入的准备工作

## 二 初始化
在使用生产组件功能之前需要调用相关api完成初始化。对外的初始化接口主要涉及以下两个类：
CreativeEngine、KSMediaPlayerInit.

初始化主要分为以下几部分，大概顺序是
1. CreativeEngine.attachBaseContext(Application)：
   这主要是给生产组件的一些变量进行赋值操作，比如Context、appId、appKey、deviceId等。
   需要注意的是这个一行配置的初始化：
   SLCCGlobalConfig config = new SLCCGlobalConfig(KPN, SECURITY_PLUGIN_APP_KEY, SECURITY_PLUGIN_WB_KEY);
   这里有三个值：KPN、SECURITY_PLUGIN_APP_KEY、SECURITY_PLUGIN_WB_KEY由StreamLake提供，需要保证正确。

2. new KSMediaPlayerInit(this)：
   这里是初始化播放器组件，用到了第一步的参数KPN和SECURITY_PLUGIN_APP_KEY，和第一步保持一致，第三个参数deviceId可以由客户自己生成，sdk也提供了生成deviceId的方法：KSMediaPlayerConfig#generateDeviceId。

3. CreativeEngine.init(Application)：
   这一步主要是初始化生产组件的各个module；

4. CreativeEngine.launchActivityCreate(Activity, Bundle)：
   这一步初始化网络监听模块以及InitManagerImpl模块

5. AppManager.get().setmEditOutputCallback(EditOutputCallback)：
   这一步是设置视频编辑的回调。在视频编辑页，完成视频编辑后，生成一个mp4产物，存储在手机sd卡里，生产组件会把改视频文件的绝对路径用String类型的变量通过该接口通知给业务层，业务层拿到该String可以做一些业务的相关处理，比如上传、发布、发动态等。



## 四 生产组件使用
生产组件主要提供三个Activity页面：
1. 拍摄录制页
   调用代码示例,CreativeEngine#startCameraActivity：
   ```
   public static void startCameraActivity(Activity activity) {
      SLCCEntranceRouter.startCameraActivity(activity);
   }
   ```
2. 相册页
   调用代码示例CreativeEngine#startAlbumActivityInternal：
   ```
   public static void startAlbumActivityInternal(Activity activity) {
      SLCCEntranceRouter.startAlbumActivityInternal(activity);
   }
   ```
3. 编辑页
   调用代码示例CreativeEngine#startEditorActivityInternal：
   ```
   public static void startEditorActivityInternal(Activity activity, String path) {
      AlbumSelectedMediasHandler mMediasHandler = new AlbumSelectedMediasHandler((FragmentActivity) activity);
      List<QMedia> qMediaList = new ArrayList<>();
      QMedia item = new QMedia(1, path, 0, 0, 1);
      qMediaList.add(item);
      mMediasHandler.setExportCount(1);
      mMediasHandler.process(qMediaList, false, false, String.valueOf(Math.random()), "videoSelect", "videoSelectActivityId", false, false);
   }
   ```

## 四 常见问题&注意事项
1. 生产组件和开源sdk：com.noober.background:core 存在冲突，解决方案是把该sdk的代码和资源复制到客户的项目中，并把冲突的资源重命名，declare-styleable命名加上noober_前缀，
2. 生产组件可能会跟客户的一些so静态库冲突，可以在app/build.gradle中使用pickFirst 'lib/*/xxx.so'解决

