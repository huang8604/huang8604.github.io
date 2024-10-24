---
tags:
  - blog
  - 测试工具
categories:
  - Android
  - ubuntu
title: CTS 在内网机器如何测试
date: 2024-09-16T04:39:38.050Z
lastmod: 2024-09-18T02:38:13.000Z
---
新版本的CTS工具，需要连接外网才能够测试，以往的版本是不需要的。

根据出错log，和源码

[Site Unreachable](https://cs.android.com/android/platform/superproject/+/master:tools/tradefederation/core/clearcut_client/com/android/tradefed/clearcut/ClearcutClient.java;drc=111c872ed55b5aed61fb905df39122913293d81d;l=133)

###### DISABLE\_CLEARCUT\_KEY

出错log

```java
java.net.SocketTimeoutException: Connect timed out
	at java.base/sun.nio.ch.NioSocketImpl.timedFinishConnect(NioSocketImpl.java:546)
	at java.base/sun.nio.ch.NioSocketImpl.connect(NioSocketImpl.java:597)
	at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:327)
	at java.base/java.net.Socket.connect(Socket.java:633)
	at java.base/sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:304)
	at java.base/sun.net.NetworkClient.doConnect(NetworkClient.java:178)
	at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:532)
	at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:637)
	at java.base/sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:266)
	at java.base/sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:380)
	at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:193)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1242)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:1128)
	at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:179)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.getOutputStream0(HttpURLConnection.java:1430)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.getOutputStream(HttpURLConnection.java:1401)
	at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.getOutputStream(HttpsURLConnectionImpl.java:220)
	at com.android.tradefed.clearcut.ClearcutClient.sendToClearcut(ClearcutClient.java:344)
	at com.android.tradefed.clearcut.ClearcutClient.lambda$flushEvents$1(ClearcutClient.java:322)

```

问题源码：

```java

/** Client that allows reporting usage metrics to clearcut. */

public class ClearcutClient {

public static final String DISABLE_CLEARCUT_KEY = "DISABLE_CLEARCUT";

/** Returns True if clearcut is disabled, False otherwise. */

@VisibleForTesting

boolean isClearcutDisabled() {

return "1".equals(System.getenv(DISABLE_CLEARCUT_KEY));

}
    /**
     * Create Client with customized posting URL and forcing whether it's internal or external user.
     */
    @VisibleForTesting
    protected ClearcutClient(String url, String subToolName) {
        mDisabled = isClearcutDisabled();

        // We still have to set the 'final' variable so go through the assignments before returning
        if (!mDisabled && isGoogleUser()) {
            mLogSource = INTERNAL_LOG_SOURCE;
            mUserType = UserType.GOOGLE;
        } else {
            mLogSource = EXTERNAL_LOG_SOURCE;
            mUserType = UserType.EXTERNAL;
        }
        if (url == null) {
            mUrl = CLEARCUT_PROD_URL;
        } else {
            mUrl = url;
        }
        mRunId = UUID.randomUUID().toString();
        mExternalEventQueue = new ArrayList<>();
        mSubToolName = subToolName;

        if (mDisabled) {
            return;
        }

```

根据代码，在测试前，需要  export  DISABLE\_CLEARCUT\_KEY=1

##### mHasServerSideConfig

出错log

```java
Caused by: java.net.ConnectException: 连接超时
	at java.base/sun.nio.ch.Net.connect0(Native Method)
	at java.base/sun.nio.ch.Net.connect(Net.java:579)
	at java.base/sun.nio.ch.Net.connect(Net.java:568)
	at java.base/sun.nio.ch.NioSocketImpl.connect(NioSocketImpl.java:588)
	at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:327)
	at java.base/java.net.Socket.connect(Socket.java:633)
	at java.base/sun.security.ssl.SSLSocketImpl.connect(SSLSocketImpl.java:304)
	at java.base/sun.security.ssl.BaseSSLSocketImpl.connect(BaseSSLSocketImpl.java:174)
	at java.base/sun.net.NetworkClient.doConnect(NetworkClient.java:183)
	at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:532)
	at java.base/sun.net.www.http.HttpClient.openServer(HttpClient.java:637)
	at java.base/sun.net.www.protocol.https.HttpsClient.<init>(HttpsClient.java:266)
	at java.base/sun.net.www.protocol.https.HttpsClient.New(HttpsClient.java:380)
	at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.getNewHttpClient(AbstractDelegateHttpsURLConnection.java:193)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect0(HttpURLConnection.java:1242)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:1128)
	at java.base/sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:179)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1665)
	at java.base/sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1589)
	at java.base/sun.net.www.protocol.https.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:224)
	at java.base/java.net.URL.openStream(URL.java:1161)
	at com.android.compatibility.common.tradefed.targetprep.DynamicConfigPusher.resolveUrl(DynamicConfigPusher.java:304)

```

问题源码：\
[Fetching Title#w5ix](https://android.googlesource.com/platform/test/suite_harness.git/+/0327a7112b3513c3c35df02ff59ef42cba53cae1%5E1..0327a7112b3513c3c35df02ff59ef42cba53cae1/)

[Fetching Title#4r2r](https://cs.android.com/android/platform/superproject/+/master:test/suite_harness/common/host-side/tradefed/src/com/android/compatibility/common/tradefed/targetprep/DynamicConfigPusher.java)

```
Add an option 'has-server-side-config' am: 6ede4f47c7

Original change: https://android-review.googlesource.com/c/platform/test/suite_harness/+/2719415

Change-Id: Ifa74a4cdcf22b6c1fe8ebaf4e848db7cf813feeb
Signed-off-by: Automerger Merge Worker <android-build-automerger-merge-worker@system.gserviceaccount.com>
```

```
    @Option(
            name = "has-server-side-config",
            description = "Whether there exists a service side dynamic config.")
    private boolean mHasServerSideConfig = true;

```

根据这个值查找了 run cts --help-all\
里面有如下说明：

```
  'dynamic-config-pusher' device options:
    --api-key            API key for for dynamic configuration. Default: AIzaSyAbwX5JRlmsLeygY2WWihpIJPXFLueOQ3U.
    --[no-]cleanup       Whether to remove config files from the test target after test completion. Default: true.
    --config-url         The url path of the dynamic config. If set, will override the default config location defined in CompatibilityBuildProvider. Default: https://androidpartner.googleapis.com/v1/dynamicconfig/suites/{suite-name}/modules/{module}/version/{version}?key={api-key}.
    --[no-]has-server-side-config
                         Whether there exists a service side dynamic config. Default: true.
    --config-filename    The module name for module-level configurations, or the suite name for suite-level configurations Default: cts.
    --target             The test target, "device" or "host" Default: HOST. Valid values: [DEVICE, HOST]
    --version            The version of the configuration to retrieve from the server, e.g. "1.0". Defaults to suite version string.
    --[no-]extract-from-resource
                         Whether to look for the local dynamic config inside the jar resources or on the local disk. Default: true.
                         
```

所以添加 --no-has-server-side-config 就可以跳过这一步了。

#### 总结

需要在终端先执行：\
export  DISABLE\_CLEARCUT\_KEY=1

跑测试用例的时候：\
run cts -d --skip-preconditions --no-has-server-side-config  -m CtsIcuTestCases -t android.icu.dev.test.timezone.TimeZoneTest#TestCanonicalID
