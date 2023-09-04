# Test Datadog Sourcemaps

An example demonstrating a pair of bugs and patches with `expo-datadog` and `@datadog/datadog-ci` in the iOS distribution build.

## Steps

This repo contains a blank Expo go project with the `expo-datadog` plugin to uploads sourcemaps to Datadog. There are two issues in the dependent packages:

- The `expo-datadog` xcode script assumes that it is being run in a `git` repo, but this isn't the case for an EAS build.
- The `@datadog/datadog-ci` script looks in the wrong place for the generated bundle sourcemap file.

## To run the build and see the errors

1. `rm -rf /tmp/eas-local-build && mkdir /tmp/eas-local-build`
2. Install `eas-cli@5.0.2` globally.
3. `yarn && yarn expo prebuild --platform ios`
4. Create a provisioning profile (ad-hoc?) with a certificate, and export these somewhere.
5. Open `./ios` in Xcode and set the team and certificates.
6. Create a `./credentials.json` file in the project root with the following paths and password filled in:

```json
{
  "ios": {
    "provisioningProfilePath": "{{IOS_PROVISIONING_PROFILE_PATH}}",
    "distributionCertificate": {
      "path": "{{IOS_DISTRIBUTION_CERTIFICATE_PATH}}",
      "password": "{{IOS_DISTRIBUTION_CERTIFICATE_PWD}}"
    }
  }
}
```

7. `DATADOG_API_KEY=<top-secret-key-goes-here> yarn build`. (This step will fail.)
8. Run `tail -n50 /tmp/eas-local-build/logs/testdatadogsourcemaps-testdatadogsourcemaps.log` to see the git error.

## To see the patch "fix"

Add the following to the scripts in `package.json`:

```diff
diff --git a/package.json b/package.json
index 06096b1..99ce8bb 100644
--- a/package.json
+++ b/package.json
@@ -2,6 +2,7 @@
   "name": "test-datadog-sourcemaps",
   "version": "1.0.0",
   "scripts": {
+    "postinstall": "patch-package",
     "build": "rm -rf /tmp/eas-local-build; EAS_LOCAL_BUILD_WORKINGDIR=/tmp/eas-local-build EAS_LOCAL_BUILD_SKIP_CLEANUP=1 eas build --platform ios --local --profile production",
     "start": "expo start --dev-client",
     "android": "expo run:android",
```

Run `yarn build` again. This will apply a monkey-patch to `expo-datadog` to `--disable-git`, and a patch to the local path of the outputted source map in `@datadog/datadog-ci`. The build should pass, and the outputted `testdatadogsourcemaps-testdatadogsourcemaps.log` file will contain:

```
Upload of ../main.jsbundle.map for bundle main.jsbundle on platform ios with project path /private/tmp/eas-local-build/build/ios
version: 1.0.0 build: 1 service: com.foobar.testdatadogsourcemaps
Please ensure you use the same values during SDK initialization to guarantee the success of the unminify process.
After upload is successful sourcemap files will be processed and ready to use within the next 5 minutes.
Uploading sourcemap ../main.jsbundle.map for JS file main.jsbundle

Command summary:
✅ Uploaded 1 sourcemap in 0.445 seconds.
```
