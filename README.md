# 概要
Androidで360°写真（またはパノラマ写真）を表示させるためのサンプルや、日本語の解説が少なかったため、備忘録としてサンプルを作成してみました。

![](https://github.com/TomiGie/Android-VrViewSample/raw/master/screenshot.gif)

# 開発準備

## Google VR SDK をプロジェクトに追加
Android Studioのターミナルから以下のコマンドで追加

```terminal:AndroidStudio/terminal
$ git clone https://github.com/googlevr/gvr-android-sdk.git
```
**【上記の方法がうまくいかない場合】**

[こちら](https://github.com/googlevr/gvr-android-sdk)よりGoogle VR SDKをダウンロードし、解凍する
解答したフォルダ（gvr-android-sdk）をプロジェクトの直下に追加する

# 開発

## setting.gradleに追記
```gradle:setting.gradle
include ':app'
include ':gvr-android-sdk/libraries:audio'
include ':gvr-android-sdk/libraries:base'
include ':gvr-android-sdk/libraries:common'
include ':gvr-android-sdk/libraries:commonwidget'
include ':gvr-android-sdk/libraries:panowidget'
include ':gvr-android-sdk/libraries:videowidget'
```

上記を追加したら**Sync Now**をする


もし、Unregistered VCS root detectedというメッセージが出てきたら、ignoreを選択する


## app/build.gradleに追記
dependenciesに以下を追加する

```gradle:app/build.gradle
compile project(':gvr-android-sdk/libraries:common')
compile project(':gvr-android-sdk/libraries:commonwidget')
compile project(':gvr-android-sdk/libraries:panowidget')
```
入力したら、**SyncNow**をする

**※minSdkVersionが19以上になっていないと、エラーが出るので注意**

## xmlにVrPanoramaViewを追加
表示したいレイアウトにVrPanoramaViewを追加する

```xml:activity_main.xml
<com.google.vr.sdk.widgets.pano.VrPanoramaView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/vr_view"
        android:scrollbars="none"/>
```

## ImageLoaderTask.java を追加
```java:ImageLoaderTask.java
public class ImageLoaderTask extends AsyncTask<AssetManager, Void, Bitmap> {
    private static final String TAG = "ImageLoaderTask";
    private final String assetName;
    private final WeakReference<VrPanoramaView> viewReference;
    private final VrPanoramaView.Options viewOptions;
    private static WeakReference<Bitmap> lastBitmap = new WeakReference<>(null);
    private static String lastName;

    @Override
    protected Bitmap doInBackground(AssetManager... params) {
        AssetManager assetManager = params[0];

        if (assetName.equals(lastName) && lastBitmap.get() != null) {
            return lastBitmap.get();
        }

        try(InputStream istr = assetManager.open(assetName)) {
            Bitmap b = BitmapFactory.decodeStream(istr);
            lastBitmap = new WeakReference<>(b);
            lastName = assetName;
            return b;
        } catch (IOException e) {
            Log.e(TAG, "Could not decode default bitmap: " + e);
            return null;
        }
    }

    @Override
    protected void onPostExecute(Bitmap bitmap) {
        final VrPanoramaView vw = viewReference.get();
        if (vw != null && bitmap != null) {
            vw.loadImageFromBitmap(bitmap, viewOptions);
        }
    }

    public ImageLoaderTask(VrPanoramaView view, VrPanoramaView.Options viewOptions, String assetName) {
        viewReference = new WeakReference<>(view);
        this.viewOptions = viewOptions;
        this.assetName = assetName;
    }
}
```

## MainActivity.javaにメソッドを追加

```java:MainActivity.java
public class MainActivity extends AppCompatActivity {

    private VrPanoramaView panoWidgetView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        panoWidgetView = (VrPanoramaView) findViewById(R.id.vr_view);
    }
    
    @Override
    public void onPause() {
        panoWidgetView.pauseRendering();
        super.onPause();
    }

    @Override
    public void onResume() {
        panoWidgetView.resumeRendering();
        super.onResume();
    }

    @Override
    public void onDestroy() {
        // Destroy the widget and free memory.
        panoWidgetView.shutdown();
        super.onDestroy();
    }

    private synchronized void loadPanoImage() {
        ImageLoaderTask task = backgroundImageLoaderTask;
        if (task != null && !task.isCancelled()) {
            // Cancel any task from a previous loading.
            task.cancel(true);
        }

        // pass in the name of the image to load from assets.
        VrPanoramaView.Options viewOptions = new VrPanoramaView.Options();
        viewOptions.inputType = VrPanoramaView.Options.TYPE_MONO;

        // use the name of the image in the assets/ directory.
        String panoImageName = "YOUR_PANORAMA_IMAGE_NAME.jpg";

        // create the task passing the widget view and call execute to start.
        task = new ImageLoaderTask(panoWidgetView, viewOptions, panoImageName);
        task.execute(getAssets());
        backgroundImageLoaderTask = task;
    }
}
```

## assetsフォルダを app/main 配下に追加
このフォルダにパノラマ画像素材を追加する


## パノラマ写真の表示がおかしい場合
パノラマ写真の設定には、

```
VrPanoramaView.Options.TYPE_MONO

VrPanoramaView.Options.TYPE_STEREO_OVER_UNDER
```
上記の2種類があるため、TYPE_MONOで表示がおかしい場合はTYPE_STEREO_OVER_UNDERに変えてみると上手く表示されるかもしれません


# 参考サイト
[GoogleCodelabs - Getting started widh VR View for Android](https://codelabs.developers.google.com/codelabs/vr_view_app_101/index.html?index=..%2F..%2Findex#1)

[GoogleCodeLabs GitHub - vr_view_app_101](https://github.com/googlecodelabs/vr_view_app_101/archive/master.zip)
