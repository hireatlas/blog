---
slug: building-a-ci-cd-pipeline-for-your-expo-app-using-eas
issue: building-a-ci-cd-pipeline-for-your-expo-app-using-eas
title: Building a CI/CD pipeline for your Expo app using EAS
#subtitle: ABC
description: ABC
date: 2024-03-04
category: guides
authors:
  - chris-rossi
published: false
---

[Expo Application Services](https://expo.dev/eas) (EAS) is a cloud-based service for building and publishing
React Native mobile apps, opinionated towards Expo apps.

Before migrating to EAS, we were required to build our mobile apps locally on Apple machines using Xcode and Android
Studio, then manually upload the compiled binaries into the Apple App Store and Google Play Store respectively (where we
could begin internal testing). This caused numerous issues, as software updates and outdated dependencies would
constantly break things in our release process.

Publishing a new mobile app version could take anywhere from an hour to multiple days, because there was no way to
predict what might go wrong and how long it would take to troubleshoot, especially in the case of cryptic
CocoaPods/Gradle errors. This had a direct impact on how many times per year we could publish new app releases.

**EAS takes care of everything.** Once we have a working development version of the app in Expo Go, it becomes trivial
to push it to EAS and let it handle the entire end-to-end build/release process.

Having a painless and consistent mobile workflow allows us to feel more confident when releasing more regular
improvements to our apps.

## Getting started

To get started with EAS, you will need to:

1. Sign up for a [free Expo account](https://expo.dev/signup).
2. Install the Expo CLI using `npm install -g eas-cli`.
3. Log in to your Expo account using `eas login`.
4. Configure the project and follow the steps using `eas build:configure`.

This final step will create a new `eas.json` file within your application.

You can find more information in the official [EAS Build documentation](https://docs.expo.dev/build/setup/).

## Configuration

All the configuration for EAS is defined within a single `eas.json` file.

After running the `eas build:configure` command above to generate this, you should have this file already. However, the
default file can still use some additional improvements.

For example, you may want to create new EAS profiles which will allow you to build with different configurations and for
different environments. Here are the profiles we currently use:

* `development`: Includes the Expo Development Client (`expo-dev-client`) and is built to run inside mobile simulators,
  which points to our development API endpoints on our local machines.
* `preview`: Effectively the same as a `production` build, but specifically built for internal testing/QA purposes (can
  be installed on real test devices).
* `preview-simulator`: Identical to `preview` except that it can be installed and tested on mobile simulators if
  needed (such as when troubleshooting a bug in the Production build).
* `production`: Built for distribution to the Apple/Google Stores, and automatically increments the build number for us.

Our current `eas.json` file looks like this:

```json
{
  // [tl! collapse:start]
  "cli": {
    "version": ">= 5.9.3",
    "appVersionSource": "remote"
  },
  // [tl! collapse:end]
  "build": {
    "base": {
      "android": {
        "image": "latest"
      },
      "ios": {
        "image": "latest"
      }
    },
    "development": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://laravel.test:8080/api/v2"
      },
      "channel": "development",
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      },
      "android": {
        "buildType": "apk"
      }
    },
    "preview": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://our-app.example.com/api/v2"
      },
      "channel": "preview",
      "distribution": "internal"
    },
    "preview-simulator": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://our-app.example.com/api/v2"
      },
      "channel": "preview",
      "developmentClient": false,
      "distribution": "internal",
      "ios": {
        "simulator": true
      },
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://our-app.example.com/api/v2"
      },
      "channel": "production",
      "distribution": "store",
      "autoIncrement": true
    }
  }
}
```

As you can see, each of our profiles extends the `base` profile (allowing us to specify which EAS build environment we
use), and can point to different API endpoints.

It's also possible to override this at runtime by using `.env` files if needed, such as temporarily pointing your
development build to the Production API for testing.

Here are some helpful links for EAS Build:

- [The full documentation for the `eas.json` file](https://docs.expo.dev/eas/json/#eas-build)
- [Managing environment variables in Expo](https://docs.expo.dev/guides/environment-variables/)

## Your first build

In this step we will use EAS to actually build a working development version of our app which can be installed into a
mobile simulator.

From within the application directory, run:

```shell
eas build -p ios --profile development
```

This will trigger an iOS build using the `development` profile, which you can also monitor via the Expo web UI:

![Creating the GitHub repository via Laravel Vapor](/assets/articles/building-a-ci-cd-pipeline-for-your-expo-app-using-eas/expo-web-ui.png)

If the build happens to fail for any reason, you can review the full XCode or Gradle logs from this screen.

When this build completes successfully, you should be prompted by EAS CLI to install it within the iOS simulator, which
you should do to verify that it was built correctly.
You can also download the compiled build artifact from the Expo web UI.

Alternatively, if you are developing on a macOS device, you can
install [Expo Orbit](https://docs.expo.dev/build/orbit/), which is a GUI to deploy builds to simulators running on your
local machine, allowing rapid testing across multiple iOS and Android devices.

Once Orbit is installed, click the "**Open with Orbit**" button on any successful build on the Expo web UI.

![Expo Web - Open with Orbit for MacOS](/assets/articles/building-a-ci-cd-pipeline-for-your-expo-app-using-eas/expo-web-open-with-orbit.png)

![Expo Orbit for MacOS](/assets/articles/building-a-ci-cd-pipeline-for-your-expo-app-using-eas/expo-orbit.png)

You can also now repeat the process for an Android build:

```shell
eas build -p android --profile development
```

## Automating your build workflow using GitHub Actions

We have now successfully built and tested a working development version of our mobile app manually using Expo CLI.

We could repeat the steps for a `production` build, however a better approach is to trigger this automatically
whenever we merge changes into our `production` (or `main`) GitHub branch.

{% callout type="note" title="Not using GitHub?" %}
Refer to the [official documentation](https://docs.expo.dev/build/building-on-ci/) for other CI systems including
Bitbucket Pipelines, GitLab CI, and Travis.
{% /callout %}

First, you will need
to [generate a Personal Access Token](https://docs.expo.dev/accounts/programmatic-access/#personal-access-tokens) for
Expo. This will allow GitHub Actions to authenticate to the Expo API and trigger builds automatically. Once you have
this token, you can add it to your GitHub Actions secrets:

![GitHub Secrets - Expo Token](/assets/articles/building-a-ci-cd-pipeline-for-your-expo-app-using-eas/github-secrets-expo-token.png)

You can then create a new GitHub Action YAML file, e.g. `.github/workflows/eas-build.yml`

```yaml
name: EAS Build
on:
  workflow_dispatch:
    inputs:
      platform:
        description: 'Which platform to build for?'
        required: true
        type: choice
        default: all
        options:
          - ios
          - android
          - all
  push:
    branches:
      - production # or `main`, or whatever you use
jobs:
  build:
    name: Install and build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18.x
          cache: npm
      - name: Setup Expo and EAS
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - name: Install dependencies
        run: npm ci
      - name: Set the variables
        run: echo "DEPLOY_PLATFORM=${{ inputs.platform || 'all' }}" >> $GITHUB_ENV
      - name: Build on EAS
        run: eas build --platform "$DEPLOY_PLATFORM" --non-interactive --no-wait --profile production
```

Whenever we merge into our `production` branch, GitHub will trigger the `Install and build` job. The final step of the
job is to trigger an EAS build of the `production` profile on all platforms (iOS and Android).

{% callout type="warning" title="Check the --no-wait flag" %}
EAS builds can take anywhere from **5 - 20 minutes** to run, so our workflow runs with the `--no-wait` flag which allows
us to immediately finish the job and save CI minutes.

If you prefer that the job continues running until the EAS build has completed, you can omit this argument.
{% /callout %}

This particular workflow can also be triggered manually via the GitHub UI if needed (across one or both platforms):

![GitHub Actions - Manually run workflow](/assets/articles/building-a-ci-cd-pipeline-for-your-expo-app-using-eas/github-run-workflow-manual.png)

Once this Action has completed you should successfully see a new production build within the Expo web UI.

If you have not yet run a production build on EAS before, you may have
to [upload your Apple/Google credentials](https://docs.expo.dev/app-signing/app-credentials/) for the app to be built
correctly.

## Releasing to production

Expo also has the ability to automatically release builds of your app to App Store Connect and Google Play Console,
where they can be distributed to internal testers via TestFlight and Test Tracks respectively.

This means that within minutes of merging changes into the `production` branch, your QA team could upgrade to the latest
version of your app, making it a completely hands-off workflow.

To do this, you first need to configure Expo's credentials for submitting to Apple/Google as per Expo's documentation:

* [Submitting to the Apple App Store](https://docs.expo.dev/submit/ios/)
* [Submitting to the Google Play Store](https://docs.expo.dev/submit/android/)

You may need to include additional settings in your `eas.json` file, such as your iOS app/team identifiers:

```json
{
  // [tl! collapse:start]
  "cli": {
    "version": ">= 5.9.3",
    "appVersionSource": "remote"
  },
  "build": {
    "base": {
      "android": {
        "image": "latest"
      },
      "ios": {
        "image": "latest"
      }
    },
    "development": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://laravel.test:8080/api/v2"
      },
      "channel": "development",
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      },
      "android": {
        "buildType": "apk"
      }
    },
    "preview": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://our-app.example.com/api/v2"
      },
      "channel": "preview",
      "distribution": "internal"
    },
    "preview-simulator": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://our-app.example.com/api/v2"
      },
      "channel": "preview",
      "developmentClient": false,
      "distribution": "internal",
      "ios": {
        "simulator": true
      },
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "extends": "base",
      "env": {
        "EXPO_PUBLIC_API_BASEURL": "https://our-app.example.com/api/v2"
      },
      "channel": "production",
      "distribution": "store",
      "autoIncrement": true
    }
  },
  // [tl! collapse:end]
  "submit": {
    "production": {
      "ios": {
        "ascAppId": "9876543210",
        "appleTeamId": "ABC12345"
      }
    }
  }
}
```

You can then modify your GitHub Action to include the `--auto-submit` parameter in the eas build command:

```yaml
name: EAS Build
on: # [tl! collapse:start]
  workflow_dispatch:
    inputs:
      platform:
        description: 'Which platform to build for?'
        required: true
        type: choice
        default: all
        options:
          - ios
          - android
          - all
  push:
    branches:
      - production
  # [tl! collapse:end]
jobs:
  build:
    name: Install and build
    runs-on: ubuntu-latest
    steps:
      # [tl! collapse:start]
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18.x
          cache: npm
      - name: Setup Expo and EAS
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - name: Install dependencies
        run: npm ci
      - name: Set the variables
        run: echo "DEPLOY_PLATFORM=${{ inputs.platform || 'all' }}" >> $GITHUB_ENV
        # [tl! collapse:end]
      - name: Build on EAS and Auto-Submit to App/Play Stores if successful
        run: eas build --platform "$DEPLOY_PLATFORM" --non-interactive --no-wait --profile production --auto-submit # [tl! **]
```

This will automatically create pending iOS and Android submissions, which will roll out as soon as the linked builds
complete successfully. You can find these in the Submissions section of the Expo web UI.

When the submission is completed, your internal testers will receive an email from Apple/Google notifying them that
there is a new release of the app to test. They can install this via TestFlight or the Google Play Store.

![New App Version Notification Email - Apple TestFlight](/assets/articles/building-a-ci-cd-pipeline-for-your-expo-app-using-eas/new-app-email-testflight.png)

![New App Version Notification Email - Google Play Console](/assets/articles/building-a-ci-cd-pipeline-for-your-expo-app-using-eas/new-app-email-google.png)

Once internal testing/QA is completed, you can use the App Store Connect and Google Play Console websites to promote
each new version to the public.

## Managing app versions

You can use `eas.json` to automatically manage your app versions instead of manually incrementing this via `app.json`.
Our example above is configured to only increment the current version when a production build is run, so that
development/preview builds don't create unnecessary versions.

If you already have an existing application, you can use the following CLI command to set Expo to your current app
versions, so that it can begin incrementing from there as of the next release:

```shell
eas build:version:set
```

More information can be found [here](https://docs.expo.dev/build-reference/app-versions/).

## Next steps

Expo includes many other features which you may find useful for improving your mobile app workflow, such as:

* [EAS Updates](https://docs.expo.dev/eas-update/introduction/): Hosted service that serves updates for projects using
  the expo-updates library.
* [EAS Metadata](https://docs.expo.dev/eas/metadata/): Automate and maintain your app store presence from the command
  line.

{% call-to-action
title="Looking for help with your Expo app?"
buttonText="Contact us today"
/%}