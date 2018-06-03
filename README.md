app自动化时，各种**不期待的弹层弹窗，升级广告**等时有飞出，由于弹窗具有**不定时，不定页面**等很多不确定性。有的弹窗很不友好，不×掉，很难进行下一步操作，造成 测试用例失败。而判断是否有弹窗，弹层很麻烦。
研究一下 appium和手机通信的原理就不难发现，运行appium时推送手机AppiumBootstrap.jar的中，有这么一段代码再**listenForever**。
#### 
 ```java
/**
 * The Bootstrap class runs the socket server.
 * 
 */
public class Bootstrap extends UiAutomatorTestCase {

  public void testRunServer() {
    Find.params = getParams();
    final boolean disableAndroidWatchers = Boolean.parseBoolean(getParams()
        .getString("disableAndroidWatchers"));
    final boolean acceptSSLCerts = Boolean.parseBoolean(getParams().getString(
        "acceptSslCerts"));

    SocketServer server;
    try {
      server = new SocketServer(4724);
      server.listenForever(disableAndroidWatchers, acceptSSLCerts);
    } catch (final SocketServerException e) {
      Logger.error(e.getError());
      System.exit(1);
    }

  }
}

那么我们可以利用这个长监听，干点点掉弹窗的事儿会不会很爽，弹窗一弹出，就把他点掉，点掉，点掉。
其实在UiWatchers.java中，作者已经写了很多UI监听了，贴一小段代码。
public class UiWatchers {
  private static final String LOG_TAG = UiWatchers.class.getSimpleName();
  private final List<String>  mErrors = new ArrayList<String>();

  /**
   * We can use the UiDevice registerWatcher to register a small script to be
   * executed when the framework is waiting for a control to appear. Waiting may
   * be the cause of an unexpected dialog on the screen and it is the time when
   * the framework runs the registered watchers. This is a sample watcher
   * looking for ANR and crashes. it closes it and moves on. You should create
   * your own watchers and handle error logging properly for your type of tests.
   */
  public void registerAnrAndCrashWatchers() {

    UiDevice.getInstance().registerWatcher("ANR", new UiWatcher() {
      @Override
      public boolean checkForCondition() {
        UiObject window = new UiObject(new UiSelector()
            .className("com.android.server.am.AppNotRespondingDialog"));
        String errorText = null;
        if (window.exists()) {
          try {
            errorText = window.getText();
          } catch (UiObjectNotFoundException e) {
            Log.e(LOG_TAG, "dialog gone?", e);
          }
          onAnrDetected(errorText);
          postHandler();
          return true; // triggered
        }
        return false; // no trigger
      }
    });

在这个类中我们加入自己的弹窗监听
  public void registerMyPopupWatcher() {
	    UiDevice.getInstance().registerWatcher("myPopup", new UiWatcher() {
	      @Override
	      public boolean checkForCondition() {

	  		//广告提示
	  		UiObject ggPop = new UiObject(new UiSelector().resourceId("com.gift.android:id/close_view"));
	  		
	  		//站点切换
	  		UiObject addChgPop = new UiObject(new UiSelector().resourceId("com.gift.android:id/bt_cancel"));
	  		
	  		//升级提示
	  		UiObject upPop = new UiObject(new UiSelector().resourceId("com.gift.android:id/close"));

	  		if (upPop.exists()) {
	  			System.out.println("you have a updateApp popup window");
	  			try {
	  				upPop.clickAndWaitForNewWindow();
	  				return true; // triggered
	  			} catch (UiObjectNotFoundException e) {
	  				Log.e(LOG_TAG, "Exception", e);
	  			}
	  		}else if (ggPop.exists()) {
	  			System.out.println("you have a popup window");
	  			try {
	  				ggPop.clickAndWaitForNewWindow();
	  				return true; // triggered
	  			} catch (UiObjectNotFoundException e) {
	  				Log.e(LOG_TAG, "Exception", e);
	  			}
	  		}else if (addChgPop.exists()) {
	  			System.out.println("you have a change site popup window");
	  			try {
	  				addChgPop.clickAndWaitForNewWindow();
	  				return true; // triggered
	  			} catch (UiObjectNotFoundException e) {
	  				Log.e(LOG_TAG, "Exception", e);
	  			}
	  		}
	        return false; // no trigger
	      }
	    });

	    Log.i(LOG_TAG, "Registered myPopupPopup Watchers");
	  }
在listenForever的方法中去调用一下，这样我们也监听Forever了。

写完编译成AppiumBootstrap.jar包，放在\Appium\node_modules\appium\build\android_bootstrap\下面。随着appium服务启动时，推送到手机中。
跑一下看效果。
app中弹窗中有如下resourceId的，统统被点掉了。
resourceId="com.gift.android:id/close_view";
resourceId="com.gift.android:id/bt_cancel";
resourceId="com.gift.android:id/close";
