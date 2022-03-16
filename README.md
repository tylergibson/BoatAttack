**_Note:This repository uses GitLFS, to use this repo you need to pull via Git and make sure GitLFS is installed locally_**

# Boat Attack - GitHub Actions + GameCI + Fastlane Example

This repo is intended to be a working example of how you can use GitHub Actions to build a non-trivial Unity game for iOS and Android, including uploading builds to Apple TestFlight or App Store submission and using git-lfs for assets. If you follow the steps in this README, it will completely manage the codesigning process without requiring access to Mac hardware.

The underlying game itself is a [sample project provided by Unity](https://github.com/unity-technologies/boatattack). For more information, check out the original project's README.

**For now, this setup is extremely manual and still a rough draft.** I will be working on a version of this that is more highly automated, likely publishing some GitHub Actions you can consume in your workflows. But for now, this is a relatively involved process, and potentially one that is missing a few steps in this documentation.
 
## Setting up Unity builds
If you follow these (exhaustive!) steps, you will have a functioning set of GitHub Actions workflows capable of building your Unity game for Windows, Mac, Linux, Android, and iOS, including separate workflows to upload iOS builds to Apple for TestFlight distribution or App Store submission. 

All of the instructions that follow are based around configuring your repo to build, code-sign, and upload iOS builds. If you're not interested in iOS builds, I'd recommend just checking out the standard [GameCI documentation](https://game.ci).

**If you're hoping to apply this workflow to your existing Unity game**, follow all of these steps. The next section will have you copy the relevant files from this repo.

**If you're just poking around at this repo or want to use it as a jumping-off point**, you can fork this repo as-is. You can skip the next section ("Set up Ruby"), but will need to complete the rest of the setup sections.

### Set up Ruby in your repo and copy over files
You can skip this step if you've forked this repo instead of setting up your own existing project.

This project relies on [Fastlane](https://fastlane.tools) for iOS and Android building and codesigning. Since Fastlane is a Ruby-based tool, we need to start by doing a little Ruby setup.

You'll need to have your GitHub repo open in an environment where you have a bash shell and a valid version of Ruby. This can be a local computer, but I've also had success running these steps in the browser via a GitHub Codespaces instance, which will have Ruby installed by default.

1. Make sure you have Bundler installed. If the `bundle` command does nothing, run `gem install bundler`

2. Copy the `Gemfile` from this repo into your repo

3. Run `bundle lock --add-platform x86_64-linux x86_64-darwin-19`. This makes sure that your Ruby setup will work on both Linux and MacOS, which is important since we'll be running it on both on GitHub Actions.

4. Copy the `fastlane` folder from this repo into your repo. It should have one file, a `Fastfile`.

5. Copy the `.github` folder from this repo into your repo.


### Set Up A Code-Signing Repo
This project uses [Fastlane Match](https://codesigning.guide) to store your Apple code-signing certificates and provisioning profiles in an encrypted git repo. The alternative is manually storing and passing around certificate files; this makes things much easier. 

After successfully following all of these steps, you will have an encrypted git repo containing a valid code-signing certificate and provisioning profile for your game, which is what we will need to automatically code-sign your game and upload it to Apple all using GitHub Actions (in the next section).

1. Create a new private repo on GitHub.

2. In your project repo, add a Repository Secrets by going to Settings -> Secrets and clicking the "New repository secret" button in the top-right. It should be called `MATCH_REPOSITORY`, and its value should be the GitHub user and repo name of your project (e.g. if your Match repo is https://github.com/someuser/my_match_repo, you would enter the value `someuser/my_match_repo`).


3. Add another Repository Secret with the key `MATCH_PASSWORD` and a secure password as the value. I recommend using a random password generator such as is included with most password managers. You'll want to note this password, as you will need it if you ever want to manually access your iOS certificates. Once entered as a Repository Secret, GitHub will not let you view the value, so you'll need to store it securely somewhere else.

4. When Fastlane initially sets up your repo and certificates, it will generate a deploy token and write that as a secret to your project repo. In order to do that, you'll need to generate a GitHub Personal Access Token with the permission to do that. Go to https://github.com/settings/tokens, click "Generate new token". It needs all "repo" permissions. Give it a suitable name, as well as choosing if you want your token to expire. After doing this and clicking "Generate token", GitHub will show you your token. Create another Repository Secret in your main repo called `GH_PAT` with your token as the value. If you choose to have your token expire, whenever you regenerate a new token you'll just need to update the value of that Secret.

5. Next, you will need to generate Apple App Store Connect keys. Go to https://appstoreconnect.apple.com/access/users and log in. Go to the "Keys" tab and click the plus sign (+) to generate a new set of keys. Enter a name and select "Developer" access. Once it's been generated, click the "Download API key" link, which will download a file. Note you can only do this once. Add three Repository Secrets: `APP_STORE_CONNECT_KEY_ID` should be the Key ID that shows up in the table row for your newly-generated key, `APP_STORE_CONNECT_ISSUER_ID` should contain the issuer ID displayed at the top of the page, and `APP_STORE_CONNECT_KEY` should include the full text of the file you downloaded. This will start with ```-----BEGIN PRIVATE KEY-----``` and end with ```-----END PRIVATE KEY-----```.

6. If you haven't already registered an App ID with Apple for your game, go to https://developer.apple.com/account/resources/identifiers/list and do so. Choose whatever options make sense for you, but you'll want to create an explicit bundle ID instead of wildcard. Once it has been created, add another Repository Secret called `BUNDLE_ID` containing your game's bundle ID. For now, this workflow assumes your iOS and Android bundle names are identical; you may need to do some manual tweaking if that's not the case for your game. (TODO: This can also be automated using Fastlane if you have access to a Mac, but I haven't documented that flow yet.)

7. Now that we've done all this work, we can run our setup GitHub Action workflow! Go to the "Actions" tab in the GitHub repo, and run the "iOS One-Time Setup" workflow name by click on the workflow, clicking the "Run Workflow" button, and then clicking the second "Run Workflow" button after confirming you're running it on the git branch you've just done all this configuration on. 

### Building Your iOS and Android App
Now that we've done all that setup work, and generated code-signing certificates for iOS, we can actually build your game for iOS and Android! By the end of this section, you will have successfully run a GitHub Actions workflow to build your game for iOS and Android, code-sign your iOS build, and upload that build to Apple for TestFlight or App Store distribution.

1. Add three more Repository Secrets: `APP_NAME`, `BUILD_NUMBER`, and `VERSION`, containing the user-friendly display name for your game and the current version and build numbers (TODO: Pull this directly from Unity config files instead of requiring people to manually enter it).

2. If you have not already, create a new App entry for your game on [App Store Connect](https://appstoreconnect.apple.com/apps), making sure to select the same bundle ID you're referencing in your config.

3. Assuming you have a Unity Pro license, create three Repository Secrets: `UNITY_EMAIL`, `UNITY_PASSWORD`, and `UNITY_SERIAL`, containing your Unity login credentials (email address and password) as well as your Serial Key, which can be found at https://id.unity.com/en/subscriptions if logged in. (TODO: Add instructions for Personal licenses. This repo already contains a workflow for the manual activation step.)

4. As written, the GitHub Actions workflows in this repo will build for iOS and Android, including uploading the resulting Android APK as a build artifact. If you want to change this, open `.github/workflows/build.yml`. Near the top of the file will be a section that reads as follows:
```
    strategy:
      matrix:
        targetPlatform:
          - iOS
          - Android
```
Feel free to modify this as desired by adding or removing any number of [supported platforms](https://docs.unity3d.com/ScriptReference/BuildTarget.html). Note that this build step runs on Linux, so you are limited to Unity builds that can be built on Linux machines (i.e. not most big commercial game consoles). While iOS does require Mac hardware to build, our build setup automatically runs the appropriate Xcode build step on Mac hardware for you without any additional configuration needed.

4. Go to the Actions tab of your repo and run the "Unity Build" workflow. This may take a while (anywhere from tens of minutes to hours). If it completes successfully, you should see a number of build artifacts: for Android and most other non-iOS platforms, this should be executable files (e.g. an APK), and for iOS it will be a zip file containing an Xcode project. If the build was successful, it did in fact successfully compile the Xcode project into a IPA file, but that currently isn't exposed as an artifact.

5. To specifically build your iOS app and upload it to TestFlight, go to the Actions tab of your repo and run the "TestFlight Build and Deploy" workflow. This will compile the Unity project into an Xcode project and compile the Xcode project on Mac hardware (as in the "Unity Build" workflow), but then it will also codesign your app, upload it to Apple, and mark it as a TestFlight build. Note that, once this workflow is completed, it still may take some time for your build to show up in App Store Connect, as Apple can take an indeterminate amount of time (frequently minutes to hours) to process your build. If you want the workflow to not be marked as "completed" until the build has finished processing on Apple's end, you can change the `skip_waiting_for_build_processing` flag in the `testflight_beta` lane of the `Fastfile`, but note that if Apple takes too long this might cause the build to time out and be marked as a failing build (despite successfully uploading to Apple).  

6. If you are trying to submit a build to the App Store, there is also an "App Store Build and Deploy" workflow that works the same way as the TestFlight one. (TODO: I've tested this on other similar projects, but not on a stock copy of this repo.)


## Roadmap
There are a few big items I'm hoping to add to this in the short term
* Run through these steps on a brand new fork of this project as well as an existing unrelated Unity project, confirming they work as expected
* Potentially add full support for uploading to the Google Play store, although this is slightly involved since you need to manually upload at least one build to Google before they will let you automate it
* Abstract out a lot of the setup, automating a few more steps than currently are and likely extracting at least some of this into third-party GitHub Actions that developers can consume from their own workflows.
* More clearly document the current limitations to GitHub Actions that may get in your way when using this workflow on larger-scale projects
* Write out more about how Git-LFS intersects with this, and confirm whether this repo will work as-is when forked if you don't have any GitHub git-lfs data packs.

**If you've made it this far, thanks for your interest!** This is very much a work-in-progress, and I'd love your feedback. If you're trying to set this up and are running into issues, feel free to open up a GitHub Issue here and I'd legitimately love to help you solve it so I can iron out the kinks in this setup process.
