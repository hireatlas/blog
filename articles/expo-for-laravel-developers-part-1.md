---
slug: expo-for-laravel-developers-part-1
issue: expo-for-laravel-developers-part-1
title: "Expo for Laravel Developers - Building your first app"
description: Pipe lines
date: 2024-05-03
category: guides
authors:
  - chris-rossi
  - mitchell-davis
published: false
---

Building mobile apps used to be hard. But now, thanks to frameworks like [Expo](https://expo.dev/), 
this process is easier than ever! 

Here at Atlas we love working with both Laravel and Expo, so in this post we'll try to share 
just how easy it can be for developers to build a working mobile app (compatible with both iOS and Android) with Expo.
Our app will use Laravel as a simple backend API. 

## Laravel API Backend

To keep things simple, we have installed a boilerplate Laravel app and defined the following route in `api.php`: 

```php
Route::get('time', function (Request $request) {
    return response()->json([
        // Return the current server time
        'time' => now()->toDateTimeString(),
    ]);
});
```

You should be able to access this via: `http://laravel.test/api/time` (where `laravel.test` is your Laravel server address).
This should give you the following output:
```json
{"time":"2024-05-03 03:46:26"}
```

## Getting started with Expo

The first step to building a mobile app with Expo is to install the required dependencies for a typical 
Node.js project. Specifically you'll need an LTS version of `nodejs`, with the ability to run `npm` and `npx`.

You can follow the official [Expo Installation Instructions here](https://docs.expo.dev/get-started/installation/) if you prefer.

Once this is ready, you can run the following command to create a new Expo app called `expo-todo-app`:
```shell
npx create-expo-app expo-todo-app
```

![Expo App Setup CLI Command Output](/expo-for-laravel-developers-part-1/expo-app-setup-cli-output.png)

You should now have a working boilerplate app within the newly created `expo-todo-app` subdirectory.

You can run the following commands to build the app and start the Expo Dev Server:
```shell
cd expo-todo-app
npx expo start
```

This should result in the following output:
![Expo App Start CLI Command Output](/expo-for-laravel-developers-part-1/expo-app-running-command-output.png)

We are now ready to launch the app in **Expo Go** and start testing, but how you proceed next will depend on your 
specific environment.

{% callout type="note" title="Expo Go?" %}
**Expo Go** is a mobile app which allows you to test your Expo app during development.
It is supported by both iOS and Android, both on physical devices and within simulators/emulators.
Expo Go features debugging and profiling tools, and hot-reloading when changes are made to the application codebase.

[Learn more in the official Expo Go Documentation](https://docs.expo.dev/get-started/expo-go/)
{% /callout %}
#### **macOS**
If you are developing on macOS and have [XCode](https://developer.apple.com/xcode/) installed (with working iOS Simulators), you can simply 
hit the `i` key to launch the iOS Simulator. This will install Expo Go and launch your app within it.

If you have [Android Studio](https://developer.android.com/studio) installed you can also hit the `a` key to launch Expo Go within one of your Android Emulators. 

#### **Windows/Linux**
Unfortunately iOS Simulators are only available on macOS, but you can still test Android apps using 
the Emulators included with [Android Studio](https://developer.android.com/studio).

Simply launch your preferred Emulator, then within the terminal for the Expo Dev Server hit the `a` key.
This will install Expo Go and launch your app within it.

#### **Physical Devices**
If you have a physical iOS device but are unable to work on a macOS environment, or if you require
access to device features which are unavailable in Simulators (such as the iOS camera), you can install
Expo Go on your mobile device and launch your mobile app within it during development.
As you make changes to the code, Expo Go will automatically refresh the app on your device.
However, you will be required to be on the same Wi-Fi network as your Expo Dev Server in order to connect.

To do this, visit the App/Play Store on your device and install the official "Expo Go" app.
Once installed, you can use your camera to scan the QR code shown earlier by the Expo Dev Server and launch the app in Expo Go.
If this does not work, you can launch Expo Go and manually enter the `exp://ip-address:port` server address.

![Expo Go QR Code](/expo-for-laravel-developers-part-1/expo-app-running-qr-code.png)


You should now have the app running within Expo Go.
If you try editing the example text in `App.js`, you should see the app refresh **instantly** to show your changes.

![Expo App.js boilerplate running in Expo Go](/expo-for-laravel-developers-part-1/expo-appjs-boilerplate-app.png)

You can also hit the `m` key from the terminal, or shake the physical device, to trigger the Expo Go Developer Menu. 
This provides access to additional debugging and profiling tools:

![Expo Go Developer Menu](/expo-for-laravel-developers-part-1/expo-go-dev-menu.png)

## Connecting to Laravel

To confirm that our new Expo mobile app can communicate with our Laravel API, we will replace the entire 
contents of `App.js` with the following test code:
```javascript
import React, {useEffect} from "react"
import { StatusBar } from 'expo-status-bar';
import {Button, StyleSheet, Text, View} from 'react-native';

export default function App() {
    const [currentTime, setCurrentTime] = React.useState(null);

    const fetchCurrentTime = async () => {
        setCurrentTime(null);
        /**
         * NOTE: This will be replaced by the URL in your .env.local file
         */
        const apiUrl = process.env.EXPO_PUBLIC_API_URL;
        
        try {
            const result = await fetch(apiUrl, {
                method: 'GET',
                headers: {
                    Accept: 'application/json',
                    'Content-Type': 'application/json',
                },
            });

            const data = await result.json();

            setCurrentTime(data.time);
        } catch (error) {
            alert(error);
        }
    }

    useEffect(() => {
        fetchCurrentTime();
    }, []);


    return (
        <View style={styles.container}>
            <Text>Current Server Time: { currentTime ?? 'Loading' }</Text>

            <Button onPress={() => fetchCurrentTime()}
                    title={'Refresh'}
            />

            <StatusBar style="auto"/>
        </View>
    );
}

const styles = StyleSheet.create({
    container: {
        flex: 1,
        backgroundColor: '#fff',
        alignItems: 'center',
        justifyContent: 'center',
    },
});
```

Then we'll create a new `.env.local` file in the root folder of our Expo app containing the following:
```shell
# NOTE: Replace this URL with the absolute URL for your Laravel API.
# eg 'http://laravel.test/api/time' or 'http://localhost/api/time'
EXPO_PUBLIC_API_URL="http://localhost:8080/api/time"
```

If this works, you should see the current server time displayed within the app. You can click "Refresh"
to refresh the time from the server. This confirms that your Expo Go app is able to talk with your Laravel API.

![Expo App with Laravel API Call](/expo-for-laravel-developers-part-1/expo-app-with-laravel-api-call.png)

{% callout type="note" title="Unable to connect to Laravel?" %}
When using Expo Go on a physical device, or from a Simulator, you may not be able to
connect to your Laravel API directly via its hostname (eg `laravel.test` or `localhost`) due to DNS resolution.

You can try changing the API URL to your LAN IP address (eg `http://192.168.1.1:8080/api/time`), 
or you can use tools like [Laravel Sail Share](https://laravel.com/docs/11.x/sail#sharing-your-site),
[Laravel Valet Expose](https://laravel.com/docs/11.x/valet#sharing-sites-via-expose), 
or [ngrok](https://ngrok.com/) to setup a tunnel for testing.
{% /callout %}

## Environment Files
Expo fully supports dotenv files, so it is best practice to use these to store URLs and any other environment-specific values or secrets here.
This allows you to swap them out at runtime, for example allowing your app to connect to a staging API during development or testing.

You can find more information about how this works in the Expo Documentation:

[Environment variables in Expo](https://docs.expo.dev/guides/environment-variables/)

## Additional Packages
Our demo app is able to perform simple API calls back to our server, but typically mobile apps require many
more features than this. Some common requirements include:
* Push notifications
* Taking photos
* GPS/Geofencing

Luckily the Expo ecosystem includes a wide range of additional packages which allow us to implement these
features in a consistent manner, without having to waste time on hardware-specific or OS-specific quirks.

You can find a list of these here:
[Expo SDK Documentation](https://docs.expo.dev/versions/latest/)

Some of our favourite Expo packages include:

| Package Name                                                                         | Description                                                                                                                       |
|--------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| [Expo Camera](https://docs.expo.dev/versions/latest/sdk/camera-next/)                | Preview the front/rear camera from within your app, take photos/videos, and scan barcodes                                         |
| [Expo Location](https://docs.expo.dev/versions/latest/sdk/location/)                 | Access the GPS location of the device, and setup geofencing listeners which trigger when the device enters/leaves a specific area |
| [Expo Notifications](https://docs.expo.dev/versions/latest/sdk/notifications/)       | Subscribe the device to receive push notifications                                                                                |
| [Expo ImageManipulator](https://docs.expo.dev/versions/latest/sdk/imagemanipulator/) | Crop, Resize, Rotate, and Flip images on the filesystem                                                                           |



## Publishing and Releasing your Mobile App

Before we can release our app to any of the App/Play stores, we need to compile it into an app package (`.ipa` for iOS, `.apk` for Android).

There are two main ways to do this with Expo.

#### **Local app compilation**
The first option is to compile the app locally.
You will require XCode installed for iOS apps (available on macOS only), and Android Studio for Android apps. You will need to 
ensure these dependencies are kept up to date as this may block you from successfully releasing updates.

Expo have documented this here: 

[Expo - Local app compilation](https://docs.expo.dev/guides/local-app-development/#local-app-compilation)

#### **Expo EAS**
The second option is to use Expo's cloud-based EAS service to compile the app for both platforms.
This is as simple as pushing your code and waiting for the build to complete. When it finishes you should have
`.ipa` and `.apk` files available which can be uploaded to Apple Testflight and/or Google Play Console.

EAS even allows you to run separate builds for Internal Testers vs Production, and submit those builds directly
to Apple/Google automatically when the build finishes.

We have documented how set this up in our blog post here:

[Building a CI/CD Pipeline for your Expo App using EAS](/blog/building-a-ci-cd-pipeline-for-your-expo-app-using-eas)

## Next Steps
Now that we have a working mobile app which can talk to Laravel's API, we will expand on this in our next post
by building a simple "Todo" app which leverages Sanctum Authentication and multiple separate API requests.
Users will be able to sign in and out of the app, lists their tasks, create new tasks, and mark tasks as completed.

To be notified when Part 2 of this series is released, please subscribe to our mailing list below.