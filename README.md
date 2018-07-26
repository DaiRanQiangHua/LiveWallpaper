# LiveWallpaper
![](https://img.shields.io/badge/platform-ios-lightgrey.svg)  ![](https://img.shields.io/badge/language-swift-orange.svg)  [![codebeat badge](https://codebeat.co/badges/77ca5356-df91-4ab1-bbfc-930897948f19)](https://codebeat.co/projects/github-com-coolspan-livewallpaper-master)  ![](https://img.shields.io/badge/license-Apache-000000.svg)  [![](https://img.shields.io/badge/CSDN-@qxs965266509-green.svg)](https://blog.csdn.net/qxs965266509/article/details/61925652)

CSDN:https://blog.csdn.net/qxs965266509/article/details/61925652


功能：设置静态壁纸和动态壁纸

1.选择系统图片作为壁纸

2.使用资源文件设置壁纸

3.使用Bitmap设置壁纸

4.使用序列帧或组合图片做动态壁纸，可设置时间自动切换壁纸，类似图片幻灯片的感觉

5.清除壁纸，自动恢复成系统默认壁纸


### 效果图：

![动态壁纸](https://github.com/coolspan/LiveWallpaper/blob/master/screenshots/livewallpaper.gif)

### 权限
```
<uses-permission android:name="android.permission.SET_WALLPAPER" />
<uses-permission android:name="android.permission.WRITE_SETTINGS" />
<uses-feature android:name="android.software.live_wallpaper" />
```

### 选择系统图片作为壁纸

```
Intent intent = new Intent(Intent.ACTION_SET_WALLPAPER);
startActivityForResult(Intent.createChooser(intent, "选择壁纸"), REQUEST_CODE_SELECT_SYSTEM_WALLPAPER);
```
使用RequestCode,可以在onActivityResult中判断是否设置成功或失败

### 使用资源文件设置壁纸
```
WallpaperManager wallpaperManager = WallpaperManager.getInstance(this);
try {
    wallpaperManager.setResource(R.raw.wallpaper);
} catch (IOException e) {
    e.printStackTrace();
}
```

### 使用Bitmap设置壁纸
```
WallpaperManager wallpaperManager = WallpaperManager.getInstance(this);
try {
    Bitmap wallpaperBitmap = BitmapFactory.decodeResource(getResources(), R.raw.girl);
    wallpaperManager.setBitmap(wallpaperBitmap);

    // 已过时的Api
    // setWallpaper(wallpaperBitmap);
    // setWallpaper(getResources().openRawResource(R.raw.girl));
} catch (IOException e) {
    e.printStackTrace();
}
```

### 动态壁纸
首先需要创建一个Service，继承于WallpaperService，如下：
```
public class LiveWallpaperService extends WallpaperService {

    private Context context;

    private LiveWallpaperEngine liveWallpaperEngine;

    private final static long REFLESH_GAP_TIME = 1000L;//如果想播放的流畅，需满足1s 16帧   62ms切换间隔时间

    @Override
    public Engine onCreateEngine() {
        this.context = this;
        this.liveWallpaperEngine = new LiveWallpaperEngine();
        return this.liveWallpaperEngine;
    }

    private class LiveWallpaperEngine extends LiveWallpaperService.Engine {

        private Runnable viewRunnable;
        private Handler handler;

        private LiveWallpaperView liveWallpaperView;

        private final SurfaceHolder surfaceHolder;

        public LiveWallpaperEngine() {
            this.surfaceHolder = getSurfaceHolder();
            this.liveWallpaperView = new LiveWallpaperView(LiveWallpaperService.this.getBaseContext());
            this.liveWallpaperView.initView(surfaceHolder);
            this.handler = new Handler();
            this.initRunnable();
            this.handler.post(this.viewRunnable);
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.ICE_CREAM_SANDWICH_MR1) {//ICE_CREAM_SANDWICH_MR1  15
                return;
            } else {
                setOffsetNotificationsEnabled(true);
            }
        }

        private void initRunnable() {
            this.viewRunnable = new Runnable() {
                @Override
                public void run() {
                    LiveWallpaperEngine.this.drawView();
                }
            };
        }

        private void drawView() {
            if (this.liveWallpaperView == null) {
                return;
            } else {
                this.handler.removeCallbacks(this.viewRunnable);
                this.liveWallpaperView.surfaceChanged(this.surfaceHolder, -1, this.liveWallpaperView.getWidth(), this.liveWallpaperView.getHeight());
                if (!(isVisible())) {
                    return;
                } else {
                    this.handler.postDelayed(this.viewRunnable, REFLESH_GAP_TIME);
                    this.liveWallpaperView.loadNextWallpaperBitmap();
                }
            }
        }

        @Override
        public void onSurfaceCreated(SurfaceHolder holder) {
            super.onSurfaceCreated(holder);
            this.drawView();
            if (this.liveWallpaperView != null) {
                this.liveWallpaperView.surfaceCreated(holder);
            } else {
                //nothing
            }
        }

        @Override
        public void onSurfaceChanged(SurfaceHolder holder, int format, int width, int height) {
            super.onSurfaceChanged(holder, format, width, height);
            this.drawView();
        }

        @Override
        public void onVisibilityChanged(boolean visible) {
            super.onVisibilityChanged(visible);
            if (this.handler != null) {
                if (visible) {
                    this.handler.post(this.viewRunnable);
                } else {
                    this.handler.removeCallbacks(this.viewRunnable);
                }
            } else {
                //nothing
            }
        }

        @Override
        public void onSurfaceDestroyed(SurfaceHolder holder) {
            super.onSurfaceDestroyed(holder);
            if (this.handler != null) {
                this.handler.removeCallbacks(this.viewRunnable);
            } else {
                //nothing
            }
            if (this.liveWallpaperView != null) {
                this.liveWallpaperView.surfaceDestroyed(holder);
            } else {
                //nothing
            }
        }

        @Override
        public void onDestroy() {
            super.onDestroy();
            if (this.handler != null) {
                this.handler.removeCallbacks(this.viewRunnable);
            } else {
                //nothing
            }
        }
    }
}
```

LiveWallpaperView:
```
public class LiveWallpaperView extends SurfaceView implements SurfaceHolder.Callback {

    private Context context;

    private Paint mPaint;
    private LruCache<String, Bitmap> mMemoryCache;

    private List<WallpaperModel> mWallpaperList;

    private WallpaperModel wallpaperModel;
    private Bitmap mNextBitmap;
    private int index;

    public LiveWallpaperView(Context context) {
        super(context);
        this.context = context;
        this.index = 0;
        this.initWallpaperList();
        this.initCacheConfig();
        this.initPaintConfig();
    }

    private void initWallpaperList() {
        this.mWallpaperList = new ArrayList<WallpaperModel>(10) {
            {
                this.add(new WallpaperModel("livewallpaper1", R.raw.livewallpaper1));
                this.add(new WallpaperModel("livewallpaper2", R.raw.livewallpaper2));
                this.add(new WallpaperModel("livewallpaper3", R.raw.livewallpaper3));
                this.add(new WallpaperModel("livewallpaper4", R.raw.livewallpaper4));
                this.add(new WallpaperModel("livewallpaper5", R.raw.livewallpaper5));
                this.add(new WallpaperModel("livewallpaper6", R.raw.livewallpaper6));
                this.add(new WallpaperModel("livewallpaper7", R.raw.livewallpaper7));
                this.add(new WallpaperModel("livewallpaper8", R.raw.livewallpaper8));
                this.add(new WallpaperModel("livewallpaper9", R.raw.livewallpaper9));
                this.add(new WallpaperModel("livewallpaper10", R.raw.livewallpaper10));
            }
        };
    }

    private void initPaintConfig() {
        this.mPaint = new Paint();
        this.mPaint.setAntiAlias(true);
        this.mPaint.setStyle(Paint.Style.STROKE);
        this.mPaint.setStrokeWidth(5);
    }

    private void initCacheConfig() {
        int maxMemory = (int) Runtime.getRuntime().maxMemory();
        this.mMemoryCache = new LruCache<String, Bitmap>(maxMemory / 8) {

            @Override
            protected int sizeOf(String key, Bitmap value) {
                return value.getRowBytes() * value.getHeight();
            }
        };
    }

    public void initView(SurfaceHolder surfaceHolder) {
        this.loadNextWallpaperBitmap();
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        holder.removeCallback(this);
        drawSurfaceView(holder);
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        drawSurfaceView(holder);
    }

    private void drawSurfaceView(SurfaceHolder holder) {
        if (this.mNextBitmap != null && !this.mNextBitmap.isRecycled()) {
            Canvas localCanvas = holder.lockCanvas();
            if (localCanvas != null) {
                Rect rect = new Rect();
                rect.left = rect.top = 0;
                rect.bottom = localCanvas.getHeight();
                rect.right = localCanvas.getWidth();
                localCanvas.drawBitmap(this.mNextBitmap, null, rect, this.mPaint);
                holder.unlockCanvasAndPost(localCanvas);
            }
        }
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        this.releaseBitmap();
    }

    public void loadNextWallpaperBitmap() {
        this.wallpaperModel = this.mWallpaperList.get(this.index);
        if (this.wallpaperModel != null) {
            Bitmap bitmap = this.mMemoryCache.get(this.wallpaperModel.getWallpaperKey());
            if (bitmap == null) {
                bitmap = BitmapFactory.decodeResource(this.getResources(), this.wallpaperModel.getWallpaperRid());
                if (bitmap != null) {
                    this.mMemoryCache.put(this.wallpaperModel.getWallpaperKey(), bitmap);
                }
            }
            if (bitmap != null) {
                this.mNextBitmap = bitmap;
            }
        }
        this.index++;
        if (this.index >= this.mWallpaperList.size()) {
            this.index = 0;
        }
    }

    /**
     * 释放动态壁纸
     */
    public void releaseBitmap() {
        try {
            if (mMemoryCache != null && mMemoryCache.size() > 0) {
                Map<String, Bitmap> stringBitmapMap = mMemoryCache.snapshot();
                mMemoryCache.evictAll();
                if (stringBitmapMap != null && stringBitmapMap.size() > 0) {
                    for (Map.Entry<String, Bitmap> entry : stringBitmapMap.entrySet()) {
                        if (entry != null && entry.getValue() != null && !entry.getValue().isRecycled()) {
                            entry.getValue().recycle();
                        }
                    }
                    stringBitmapMap.clear();
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```


接着需要注册Service到清单文件中:
```
<service
            android:name=".service.LiveWallpaperService"
            android:enabled="true"
            android:label="@string/wallpaper_name"
            android:permission="android.permission.BIND_WALLPAPER">
            <intent-filter>
                <action android:name="android.service.wallpaper.WallpaperService" />
            </intent-filter>
            <meta-data
                android:name="android.service.wallpaper"
                android:resource="@xml/my_wallpaper" />
        </service>
```

my_wallpaper:
```
<?xml version="1.0" encoding="utf-8"?>
<wallpaper xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/wallpaper_name"
    android:thumbnail="@mipmap/ic_launcher" />
```

使用方式：
```
WallpaperUtil.setLiveWallpaper(this.getApplicationContext(), MainActivity.this, MainActivity.REQUEST_CODE_SET_WALLPAPER);
```

```
public static void setLiveWallpaper(Context context, Activity paramActivity, int requestCode) {
    try {
        Intent localIntent = new Intent();
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.ICE_CREAM_SANDWICH_MR1) {//ICE_CREAM_SANDWICH_MR1  15
                          localIntent.setAction(WallpaperManager.ACTION_CHANGE_LIVE_WALLPAPER);//android.service.wallpaper.CHANGE_LIVE_WALLPAPER
        //android.service.wallpaper.extra.LIVE_WALLPAPER_COMPONENT
        localIntent.putExtra(WallpaperManager.EXTRA_LIVE_WALLPAPER_COMPONENT
                        , new ComponentName(context.getApplicationContext().getPackageName()
                                , LiveWallpaperService.class.getCanonicalName()));
        } else {
                    localIntent.setAction(WallpaperManager.ACTION_LIVE_WALLPAPER_CHOOSER);//android.service.wallpaper.LIVE_WALLPAPER_CHOOSER
        }
        paramActivity.startActivityForResult(localIntent, requestCode);
    } catch (Exception localException) {
        localException.printStackTrace();
    }
}
```

### 清除壁纸
```
WallpaperManager wallpaperManager = WallpaperManager.getInstance(this);
try {
    wallpaperManager.clear();
    // 已过时的Api
    //clearWallpaper();
} catch (IOException e) {
    e.printStackTrace();
}
```


## License

    Copyright 2018 coolspan

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
