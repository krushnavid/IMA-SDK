# IMA-SDK-Integration

## 1. Create a new Android Studio project
To create your Android Studio project, complete the following steps:
1. Start Android Studio.
2. Select Start a new Android Studio project.
3. In the Choose your project page, select the Empty Activity template.
4. Click Next.
5. In the Configure your project page, name your project and select Java for the language.Note:The IMA SDK is compatible with Kotlin. However, the example covered in this guide uses Java.
6. Click Finish.


## 2. Add the IMA SDK to your project
First, in the application-level build.gradle file, add imports for the IMA SDK to the dependencies section. Because of the size of the IMA SDK, implement and enable multidex here. This is necessary for apps with minSdkVersion set to 20 or lower. Also, add new compileOptions to specify the Java version compatibility information.
app/build.gradle

<pre>
<code>
android {
    namespace 'com.google.imasdk'
    compileSdk 34

    defaultConfig {
        applicationId "com.google.imasdk"
        minSdk 24
        targetSdk 34
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation(platform("org.jetbrains.kotlin:kotlin-bom:1.8.0"))
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.browser:browser:1.7.0'
    implementation 'androidx.media:media:1.7.0'
    implementation 'com.google.ads.interactivemedia.v3:interactivemedia:3.32.0'
}
</code>
</pre>

## 3. Add IMA SDK required permissions
Add the user permissions required by the IMA SDK for requesting ads.
app/src/main/AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.project name">

    <!-- Required permissions for the IMA SDK -->
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>


</manifest>
```


## 4. Update the app layout
Update the app's layout to include a VideoView to play both content and ads.
app/src/main/res/layout/activity_my.xml

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MyActivity"
    tools:ignore="MergeRootFrame">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:id="@+id/videoPlayerContainer" >

        <VideoView
            android:id="@+id/videoView"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </RelativeLayout>

</LinearLayout>

    
```


## 5. Import IMA into the main activity
Add the import statements for the IMA SDK. Then, update the MyActivity class to extend AppCompatActivity. The AppCompatActivity class allows for support of newer platform features on older Android devices. Then, add a set of private variables which will be used in the app.
app/src/main/java/com/example/project name/MyActivity.java

<pre>
<code>
import android.content.Context;
import android.media.AudioManager;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.view.ViewGroup;
import android.widget.MediaController;
import android.widget.VideoView;
import com.google.ads.interactivemedia.v3.api.AdDisplayContainer;
import com.google.ads.interactivemedia.v3.api.AdErrorEvent;
import com.google.ads.interactivemedia.v3.api.AdEvent;
import com.google.ads.interactivemedia.v3.api.AdsLoader;
import com.google.ads.interactivemedia.v3.api.AdsManager;
import com.google.ads.interactivemedia.v3.api.AdsManagerLoadedEvent;
import com.google.ads.interactivemedia.v3.api.AdsRenderingSettings;
import com.google.ads.interactivemedia.v3.api.AdsRequest;
import com.google.ads.interactivemedia.v3.api.ImaSdkFactory;
import com.google.ads.interactivemedia.v3.api.ImaSdkSettings;
import com.google.ads.interactivemedia.v3.api.player.VideoProgressUpdate;
import java.util.Arrays;

public class MyActivity extends AppCompatActivity {

  private static final String LOGTAG = "IMABasicSample";
  private static final String SAMPLE_VIDEO_URL =
      "https://storage.googleapis.com/gvabox/media/samples/stock.mp4";

  /**
   * IMA sample tag for a single skippable inline video ad. See more IMA sample tags at
   * https://developers.google.com/interactive-media-ads/docs/sdks/html5/client-side/tags
   */
  private static final String SAMPLE_VAST_TAG_URL =
      "https://pubads.g.doubleclick.net/gampad/ads?iu=/21775744923/external/single_ad_samples&sz=640x480&cust_params=sample_ct%3Dlinear&ciu_szs=300x250%2C728x90&gdfp_req=1&output=vast&unviewed_position_start=1&env=vp&impl=s&correlator=";

  // Factory class for creating SDK objects.
  private ImaSdkFactory sdkFactory;

  // The AdsLoader instance exposes the requestAds method.
  private AdsLoader adsLoader;

  // AdsManager exposes methods to control ad playback and listen to ad events.
  private AdsManager adsManager;

  // The saved content position, used to resumed content following an ad break.
  private int savedPosition = 0;

  // This sample uses a VideoView for content and ad playback. For production
  // apps, Android's Exoplayer offers a more fully featured player compared to
  // the VideoView.
  private VideoView videoPlayer;
  private MediaController mediaController;
  private VideoAdPlayerAdapter videoAdPlayerAdapter;

}
</code>
</pre>

## 6. Create the VideoAdPlayerAdapter class
Create a VideoAdPlayerAdapter class with the VideoView, and adapt it to IMA's VideoAdPlayer interface. This class will handle content and ad playback, and will contain the set of methods that a video player must implement to be used by the IMA SDK.
app/src/main/java/com/example/project name/VideoAdPlayerAdapter.java

<pre>
<code>
import android.media.AudioManager;
import android.media.MediaPlayer;
import android.net.Uri;
import android.util.Log;
import android.widget.VideoView;
import com.google.ads.interactivemedia.v3.api.AdPodInfo;
import com.google.ads.interactivemedia.v3.api.player.AdMediaInfo;
import com.google.ads.interactivemedia.v3.api.player.VideoAdPlayer;
import com.google.ads.interactivemedia.v3.api.player.VideoProgressUpdate;
import java.util.ArrayList;
import java.util.List;
import java.util.Timer;
import java.util.TimerTask;

/** Example implementation of IMA's VideoAdPlayer interface. */
public class VideoAdPlayerAdapter implements VideoAdPlayer {

  private static final String LOGTAG = "IMABasicSample";
  private static final long POLLING_TIME_MS = 250;
  private static final long INITIAL_DELAY_MS = 250;
  private final VideoView videoPlayer;
  private final AudioManager audioManager;
  private final List<VideoAdPlayerCallback> videoAdPlayerCallbacks = new ArrayList<>();
  private Timer timer;
  private int adDuration;

  // The saved ad position, used to resumed ad playback following an ad click-through.
  private int savedAdPosition;
  private AdMediaInfo loadedAdMediaInfo;

  public VideoAdPlayerAdapter(VideoView videoPlayer, AudioManager audioManager) {
    this.videoPlayer = videoPlayer;
    this.videoPlayer.setOnCompletionListener(
        (MediaPlayer mediaPlayer) -> notifyImaOnContentCompleted());
    this.audioManager = audioManager;
  }
}
</code>
</pre>

## 7. Override the VideoAdPlayer methods
Override the following VideoAdPlayer methods:
* addCallback()
* loadAd()
* pauseAd()
* playAd()
* release()
* removeCallback()
* stopAd()
* getVolume()
The playAd() method sets the content or ad URL and sets a listener to start playback once the media is loaded.
app/src/main/java/com/example/project name/VideoAdPlayerAdapter.java

<pre>
<code>
/** Example implementation of IMA's VideoAdPlayer interface. */
public class VideoAdPlayerAdapter implements VideoAdPlayer {


  @Override
  public void addCallback(VideoAdPlayerCallback videoAdPlayerCallback) {
    videoAdPlayerCallbacks.add(videoAdPlayerCallback);
  }

  @Override
  public void loadAd(AdMediaInfo adMediaInfo, AdPodInfo adPodInfo) {
    // This simple ad loading logic works because preloading is disabled. To support
    // preloading ads your app must maintain state for the currently playing ad
    // while handling upcoming ad downloading and buffering at the same time.
    // See the IMA Android preloading guide for more info:
    // https://developers.google.com/interactive-media-ads/docs/sdks/android/client-side/preload
    loadedAdMediaInfo = adMediaInfo;
  }

  @Override
  public void pauseAd(AdMediaInfo adMediaInfo) {
    Log.i(LOGTAG, "pauseAd");
    savedAdPosition = videoPlayer.getCurrentPosition();
    stopAdTracking();
  }

  @Override
  public void playAd(AdMediaInfo adMediaInfo) {
    videoPlayer.setVideoURI(Uri.parse(adMediaInfo.getUrl()));

    videoPlayer.setOnPreparedListener(
        mediaPlayer -> {
          adDuration = mediaPlayer.getDuration();
          if (savedAdPosition > 0) {
            mediaPlayer.seekTo(savedAdPosition);
          }
          mediaPlayer.start();
          startAdTracking();
        });
    videoPlayer.setOnErrorListener(
        (mediaPlayer, errorType, extra) -> notifyImaSdkAboutAdError(errorType));
    videoPlayer.setOnCompletionListener(
        mediaPlayer -> {
          savedAdPosition = 0;
          notifyImaSdkAboutAdEnded();
        });
  }

  @Override
  public void release() {
    // any clean up that needs to be done.
  }

  @Override
  public void removeCallback(VideoAdPlayerCallback videoAdPlayerCallback) {
    videoAdPlayerCallbacks.remove(videoAdPlayerCallback);
  }

  @Override
  public void stopAd(AdMediaInfo adMediaInfo) {
    Log.i(LOGTAG, "stopAd");
    stopAdTracking();
  }

  /** Returns current volume as a percent of max volume. */
  @Override
  public int getVolume() {
    return audioManager.getStreamVolume(AudioManager.STREAM_MUSIC)
        / audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC);
  }
</code>
</pre>


  
## 8. Set up ad tracking
In order for ad events to be registered, VideoAdPlayerCallback.onAdProgress must be called as content and ads progress. To support this, set up a timer to call onAdProgress() at a set interval.
app/src/main/java/com/example/project name/VideoAdPlayerAdapter.java

<pre>
<code>
/** Example implementation of IMA's VideoAdPlayer interface. */
public class VideoAdPlayerAdapter implements VideoAdPlayer {


  private void startAdTracking() {
    Log.i(LOGTAG, "startAdTracking");
    if (timer != null) {
      return;
    }
    timer = new Timer();
    TimerTask updateTimerTask =
        new TimerTask() {
          @Override
          public void run() {
            VideoProgressUpdate progressUpdate = getAdProgress();
            notifyImaSdkAboutAdProgress(progressUpdate);
          }
        };
    timer.schedule(updateTimerTask, POLLING_TIME_MS, INITIAL_DELAY_MS);
  }

  private void notifyImaSdkAboutAdEnded() {
    Log.i(LOGTAG, "notifyImaSdkAboutAdEnded");
    savedAdPosition = 0;
    for (VideoAdPlayer.VideoAdPlayerCallback callback : videoAdPlayerCallbacks) {
      callback.onEnded(loadedAdMediaInfo);
    }
  }

  private void notifyImaSdkAboutAdProgress(VideoProgressUpdate adProgress) {
    for (VideoAdPlayer.VideoAdPlayerCallback callback : videoAdPlayerCallbacks) {
      callback.onAdProgress(loadedAdMediaInfo, adProgress);
    }
  }

  /**
   * @param errorType Media player's error type as defined at
   *     https://cs.android.com/android/platform/superproject/+/master:frameworks/base/media/java/android/media/MediaPlayer.java;l=4335
   * @return True to stop the current ad playback.
   */
  private boolean notifyImaSdkAboutAdError(int errorType) {
    Log.i(LOGTAG, "notifyImaSdkAboutAdError");

    switch (errorType) {
      case MediaPlayer.MEDIA_ERROR_UNSUPPORTED:
        Log.e(LOGTAG, "notifyImaSdkAboutAdError: MEDIA_ERROR_UNSUPPORTED");
        break;
      case MediaPlayer.MEDIA_ERROR_TIMED_OUT:
        Log.e(LOGTAG, "notifyImaSdkAboutAdError: MEDIA_ERROR_TIMED_OUT");
        break;
      default:
        break;
    }
    for (VideoAdPlayer.VideoAdPlayerCallback callback : videoAdPlayerCallbacks) {
      callback.onError(loadedAdMediaInfo);
    }
    return true;
  }

  public void notifyImaOnContentCompleted() {
    Log.i(LOGTAG, "notifyImaOnContentCompleted");
    for (VideoAdPlayer.VideoAdPlayerCallback callback : videoAdPlayerCallbacks) {
      callback.onContentComplete();
    }
  }

  private void stopAdTracking() {
    Log.i(LOGTAG, "stopAdTracking");
    if (timer != null) {
      timer.cancel();
      timer = null;
    }
  }

  @Override
  public VideoProgressUpdate getAdProgress() {
    long adPosition = videoPlayer.getCurrentPosition();
    return new VideoProgressUpdate(adPosition, adDuration);
  }
}
</code>
</pre>



## 9. Initiate IMA in the onCreate method
Overwrite the onCreate method and add the required variable assignments to initiate IMA. In this step, create the following:
* ImaSdkSettings
* AdsLoader
This step also creates a VideoAdPlayerAdapter, a class you create later in this guide.
Finally, set up the play button to request ads, then hide when clicked.
app/src/main/java/com/example/project name/MyActivity.java

<pre>
<code>

public class MyActivity extends AppCompatActivity {

  private VideoView videoPlayer;
  private MediaController mediaController;
  private VideoAdPlayerAdapter videoAdPlayerAdapter;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_my);

    // Create the UI for controlling the video view.
    mediaController = new MediaController(this);
    videoPlayer = findViewById(R.id.videoView);
    mediaController.setAnchorView(videoPlayer);
    videoPlayer.setMediaController(mediaController);

    // Create an ad display container that uses a ViewGroup to listen to taps.
    ViewGroup videoPlayerContainer = findViewById(R.id.videoPlayerContainer);
    AudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
    videoAdPlayerAdapter = new VideoAdPlayerAdapter(videoPlayer, audioManager);

    sdkFactory = ImaSdkFactory.getInstance();

    AdDisplayContainer adDisplayContainer =
        ImaSdkFactory.createAdDisplayContainer(videoPlayerContainer, videoAdPlayerAdapter);

    // Create an AdsLoader.
    ImaSdkSettings settings = sdkFactory.createImaSdkSettings();
    adsLoader = sdkFactory.createAdsLoader(this, settings, adDisplayContainer);

 
    videoPlayer.setVideoPath(SAMPLE_VIDEO_URL);
    requestAds(SAMPLE_VAST_TAG_URL);
        
  }

}
</code>
</pre>



## 10. Add AdsLoader listeners
Add listeners for addAdErrorListener and addAdsLoadedListener. In the AdsLoadedListener, create the AdsManager, and set up the AdsManager error listener.
app/src/main/java/com/example/project name/MyActivity.java
<pre>
<code>
  @Override
  protected void onCreate(Bundle savedInstanceState) {

    // Create an AdsLoader.
    ImaSdkSettings settings = sdkFactory.createImaSdkSettings();
    adsLoader = sdkFactory.createAdsLoader(this, settings, adDisplayContainer);

    // Add listeners for when ads are loaded and for errors.
    adsLoader.addAdErrorListener(
        new AdErrorEvent.AdErrorListener() {
          /** An event raised when there is an error loading or playing ads. */
          @Override
          public void onAdError(AdErrorEvent adErrorEvent) {
            Log.i(LOGTAG, "Ad Error: " + adErrorEvent.getError().getMessage());
            resumeContent();
          }
        });
    adsLoader.addAdsLoadedListener(
        new AdsLoader.AdsLoadedListener() {
          @Override
          public void onAdsManagerLoaded(AdsManagerLoadedEvent adsManagerLoadedEvent) {
            // Ads were successfully loaded, so get the AdsManager instance. AdsManager has
            // events for ad playback and errors.
            adsManager = adsManagerLoadedEvent.getAdsManager();

            // Attach event and error event listeners.
            adsManager.addAdErrorListener(
                new AdErrorEvent.AdErrorListener() {
                  /** An event raised when there is an error loading or playing ads. */
                  @Override
                  public void onAdError(AdErrorEvent adErrorEvent) {
                    Log.e(LOGTAG, "Ad Error: " + adErrorEvent.getError().getMessage());
                    String universalAdIds =
                        Arrays.toString(adsManager.getCurrentAd().getUniversalAdIds());
                    Log.i(
                        LOGTAG,
                        "Discarding the current ad break with universal "
                            + "ad Ids: "
                            + universalAdIds);
                    adsManager.discardAdBreak();
                  }
                });
          }
        });
</code>
</pre>
        

        
## 11. Handle IMA ad events
Listen for IMA ad events with AdsManager.addAdEventListener. Using a switch statement, set up actions for the following IMA events:
* LOADED
* CONTENT_PAUSE_REQUESTED
* CONTENT_RESUME_REQUESTED
* ALL_ADS_COMPLETED
* CLICKED
The code snippet includes comments with more information on how to use the events. Once the events are set up, you can call AdsManager.init().
app/src/main/java/com/example/project name/MyActivity.java

<pre>
<code>
        adsLoader.addAdsLoadedListener(
        new AdsLoader.AdsLoadedListener() {
          @Override
          public void onAdsManagerLoaded(AdsManagerLoadedEvent adsManagerLoadedEvent) {

            adsManager.addAdEventListener(
                new AdEvent.AdEventListener() {
                  /** Responds to AdEvents. */
                  @Override
                  public void onAdEvent(AdEvent adEvent) {
                    if (adEvent.getType() != AdEvent.AdEventType.AD_PROGRESS) {
                      Log.i(LOGTAG, "Event: " + adEvent.getType());
                    }
                    // These are the suggested event types to handle. For full list of
                    // all ad event types, see AdEvent.AdEventType documentation.
                    switch (adEvent.getType()) {
                      case LOADED:
                        // AdEventType.LOADED is fired when ads are ready to play.

                        // This sample app uses the sample tag
                        // single_preroll_skippable_ad_tag_url that requires calling
                        // AdsManager.start() to start ad playback.
                        // If you use a different ad tag URL that returns a VMAP or
                        // an ad rules playlist, the adsManager.init() function will
                        // trigger ad playback automatically and the IMA SDK will
                        // ignore the adsManager.start().
                        // It is safe to always call adsManager.start() in the
                        // LOADED event.
                        adsManager.start();
                        break;
                      case CONTENT_PAUSE_REQUESTED:
                        // AdEventType.CONTENT_PAUSE_REQUESTED is fired when you
                        // should pause your content and start playing an ad.
                        pauseContentForAds();
                        break;
                      case CONTENT_RESUME_REQUESTED:
                        // AdEventType.CONTENT_RESUME_REQUESTED is fired when the ad
                        // you should play your content.
                        resumeContent();
                        break;
                      case ALL_ADS_COMPLETED:
                        // Calling adsManager.destroy() triggers the function
                        // VideoAdPlayer.release().
                        adsManager.destroy();
                        adsManager = null;
                        break;
                      case CLICKED:
                        // When the user clicks on the Learn More button, the IMA SDK fires
                        // this event, pauses the ad, and opens the ad's click-through URL.
                        // When the user returns to the app, the IMA SDK calls the
                        // VideoAdPlayer.playAd() function automatically.
                        break;
                      default:
                        break;
                    }
                  }
                });
            AdsRenderingSettings adsRenderingSettings =
                ImaSdkFactory.getInstance().createAdsRenderingSettings();
            adsManager.init(adsRenderingSettings);
          }
</code>
</pre>
          
## 12. Handle switching between ads and content
In this section, create the pauseContentForAds and resumeContent methods referenced in the previous steps. These methods will reuse the player to play both content and ads. You will need to keep track of the content position in order to resume playback following the ad break.
app/src/main/java/com/example/project name/MyActivity.java

  <pre>
<code>
/** Main activity. */
public class MyActivity extends AppCompatActivity {

  private void pauseContentForAds() {
    Log.i(LOGTAG, "pauseContentForAds");
    savedPosition = videoPlayer.getCurrentPosition();
    videoPlayer.stopPlayback();
    // Hide the buttons and seek bar controlling the video view.
    videoPlayer.setMediaController(null);
  }

  private void resumeContent() {
    Log.i(LOGTAG, "resumeContent");

    // Show the buttons and seek bar controlling the video view.
    videoPlayer.setVideoPath(SAMPLE_VIDEO_URL);
    videoPlayer.setMediaController(mediaController);
    videoPlayer.setOnPreparedListener(
        mediaPlayer -> {
          if (savedPosition > 0) {
            mediaPlayer.seekTo(savedPosition);
          }
          mediaPlayer.start();
        });
    videoPlayer.setOnCompletionListener(
        mediaPlayer -> videoAdPlayerAdapter.notifyImaOnContentCompleted());
  }
}
</code>
</pre>

## 13. Request ads
Now add the requestAds method to build an AdsRequest and use it to call AdsLoader.requestAds().
app/src/main/java/com/example/project name/MyActivity.java
  <pre>
<code>
/** Main activity. */
public class MyActivity extends AppCompatActivity {

  private void requestAds(String adTagUrl) {
    // Create the ads request.
    AdsRequest request = sdkFactory.createAdsRequest();
    request.setAdTagUrl(adTagUrl);
    request.setContentProgressProvider(
        () -> {
          if (videoPlayer.getDuration() <= 0) {
            return VideoProgressUpdate.VIDEO_TIME_NOT_READY;
          }
          return new VideoProgressUpdate(
              videoPlayer.getCurrentPosition(), videoPlayer.getDuration());
        });

    // Request the ad. After the ad is loaded, onAdsManagerLoaded() will be called.
    adsLoader.requestAds(request);
  }
}
</code>
</pre>
