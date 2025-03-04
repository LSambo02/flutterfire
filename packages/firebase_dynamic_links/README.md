# Google Dynamic Links for Firebase

[![pub package](https://img.shields.io/pub/v/firebase_dynamic_links.svg)](https://pub.dev/packages/firebase_dynamic_links)

A Flutter plugin to use the [Google Dynamic Links for Firebase API](https://firebase.google.com/docs/dynamic-links/).

With Dynamic Links, your users get the best available experience for the platform they open your link on. If a user opens a Dynamic Link on iOS or Android, they can be taken directly to the linked content in your native app. If a user opens the same Dynamic Link in a desktop browser, they can be taken to the equivalent content on your website.

In addition, Dynamic Links work across app installs: if a user opens a Dynamic Link on iOS or Android and doesn't have your app installed, the user can be prompted to install it; then, after installation, your app starts and can access the link.

For Flutter plugins for other Firebase products, see [README.md](https://github.com/FirebaseExtended/flutterfire/blob/master/README.md).

## Usage

To use this plugin, add `firebase_dynamic_links` as a [dependency in your pubspec.yaml file](https://flutter.io/platform-plugins/). You must also configure firebase dynamic links for each platform project: Android and iOS (see the example folder for details).

## Create Dynamic Links

You create a Dynamic Link either by using the Firebase console, using a REST API, iOS or Android Builder API, Flutter API, or by forming a URL by adding Dynamic Link parameters to a URI prefix specific to your app. These parameters specify the links you want to open, depending on the user's platform and whether your app is installed.

Below are instructions to create Dynamic Links using Flutter with the Firebase Dynamic Links API. This API accepts either a long Dynamic Link or an object containing Dynamic Link parameters, and returns a URL like the following example:

```
https://example.page.link/WXYZ
```

You can create a Dynamic Link programmatically by setting the following parameters and using the `DynamicLinkParameters.buildUrl()` method.

```dart
final DynamicLinkParameters parameters = DynamicLinkParameters(
  uriPrefix: 'https://abc123.app.goo.gl',
  link: Uri.parse('https://example.com/'),
  androidParameters: AndroidParameters(
      packageName: 'com.example.android',
      minimumVersion: 125,
  ),
  iosParameters: IosParameters(
      bundleId: 'com.example.ios',
      minimumVersion: '1.0.1',
      appStoreId: '123456789',
  ),
  googleAnalyticsParameters: GoogleAnalyticsParameters(
      campaign: 'example-promo',
      medium: 'social',
      source: 'orkut',
  ),
  itunesConnectAnalyticsParameters: ItunesConnectAnalyticsParameters(
    providerToken: '123456',
    campaignToken: 'example-promo',
  ),
  socialMetaTagParameters:  SocialMetaTagParameters(
    title: 'Example of a Dynamic Link',
    description: 'This link works whether app is installed or not!',
  ),
);

final Uri dynamicUrl = await parameters.buildUrl();
```

To create a short Dynamic Link, build `DynamicLinkParameters` the same way, but use the `DynamicLinkParameters.buildShortLink()` method.

```dart
final ShortDynamicLink shortDynamicLink = await parameters.buildShortLink();
final Uri shortUrl = shortDynamicLink.shortUrl;
```

To shorten a long Dynamic Link, use the DynamicLinkParameters.shortenUrl method.

```dart
final ShortDynamicLink shortenedLink = await DynamicLinkParameters.shortenUrl(
  Uri.parse('https://example.page.link/?link=https://example.com/&apn=com.example.android&ibn=com.example.ios'),
  DynamicLinkParametersOptions(ShortDynamicLinkPathLength.unguessable),
);

final Uri shortUrl = shortenedLink.shortUrl;
```

## Handle Received Dynamic Links

You can receive a Dynamic Link containing a deep link that takes the user to specific content within your app:

1. In the [Firebase Console](https://console.firebase.google.com), open the Dynamic Links section.
  - Accept the terms of service if you are prompted to do so.
  - Take note of your project's Dynamic Links URL prefix, which is displayed at the top of the Dynamic Links page. You need your project's Dynamic Links URL prefix to programmatically create Dynamic Links. Unless you are using a custom domain, a Dynamic Links URL prefix looks like `https://YOUR_SUBDOMAIN.page.link`.

Receiving dynamic links on *iOS* requires a couple more steps than *Android*. If you only want to receive dynamic links on *Android*, skip to step 5. You can also follow a video on the next two steps [here.](https://youtu.be/KLBjAg6HvG0?t=60)

2. In the **Info** tab of your *iOS* app's Xcode project:
  - Create a new **URL Type** to be used for Dynamic Links.
  - Set the **Identifier** field to a unique value and the **URL Schemes** field to be your bundle identifier, which is the default URL scheme used by Dynamic Links.

3. In the **Capabilities** tab of your app's Xcode project, enable **Associated Domains** and add the following to the **Associated Domains** list:

```
applinks:YOUR_URL_PREFIX
```

Remember not to include `https://` or any slashes or paths in your prefix

4. **If you are using a custom domain** create an key in your `Info.plist` file called `FirebaseDynamicLinksCustomDomains` and set it to your app's Dynamic Link URL prefixes. For example:

```xml
<key>FirebaseDynamicLinksCustomDomains</key>
<array>
  <string>https://example.com/promos</string>
  <string>https://example.com/links/share</string>
</array>
```
5. If you want to open android app directly without redirecting to Chrome or Any other Browser you can put an `intent-filter` in your `AndroidManifest.xml` file. It will open the system dialogue to open link with your app or other browsers. Users can then directly choose to open the link in your app.

Note: This step is optional and in case we do not implement this then link will open in chrome at first and then will eventually open your application.
```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <!-- Accepts URIs that begin with YOUR_SCHEME://YOUR_HOST -->
  <data
    android:scheme="https"
    android:host="YOUR_SUBDOMAIN.page.link" />
</intent-filter>
```
6. To receive a dynamic link, call the `getInitialLink()` method from `FirebaseDynamicLinks` which gets the link that opened the app (or null if it was not opened via a dynamic link)
and configure listeners for link callbacks when the application is active or in background calling `onLink`.

```dart
void main() {
  runApp(MaterialApp(
    title: 'Dynamic Links Example',
    routes: <String, WidgetBuilder>{
      '/': (BuildContext context) => MyHomeWidget(), // Default home route
      '/helloworld': (BuildContext context) => MyHelloWorldWidget(),
    },
  ));
}

class MyHomeWidgetState extends State<MyHomeWidget> {
  .
  .
  .
  @override
  void initState() {
    super.initState();
    this.initDynamicLinks();
  }

  void initDynamicLinks() async {
    FirebaseDynamicLinks.instance.onLink(
      onSuccess: (PendingDynamicLinkData? dynamicLink) async {
        final Uri? deepLink = dynamicLink?.link;

        if (deepLink != null) {
          Navigator.pushNamed(context, deepLink.path);
        }
      },
      onError: (OnLinkErrorException e) async {
        print('onLinkError');
        print(e.message);
      }
    );
    
    final PendingDynamicLinkData? data = await FirebaseDynamicLinks.instance.getInitialLink();
    final Uri? deepLink = data?.link;

    if (deepLink != null) {
      Navigator.pushNamed(context, deepLink.path);
    }
  }
  .
  .
  .
}
```

If your app did not open from a dynamic link, `getInitialLink()` will return `null`.

## Getting Started

See the `example` directory for a complete sample app using Google Dynamic Links for Firebase.

## Issues and feedback

Please file FlutterFire specific issues, bugs, or feature requests in our [issue tracker](https://github.com/FirebaseExtended/flutterfire/issues/new).

Plugin issues that are not specific to Flutterfire can be filed in the [Flutter issue tracker](https://github.com/flutter/flutter/issues/new).

To contribute a change to this plugin,
please review our [contribution guide](https://github.com/FirebaseExtended/flutterfire/blob/master/CONTRIBUTING.md)
and open a [pull request](https://github.com/FirebaseExtended/flutterfire/pulls).
