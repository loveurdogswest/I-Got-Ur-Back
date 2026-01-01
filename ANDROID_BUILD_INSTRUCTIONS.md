# Android AAB Build Instructions for Play Store

## Package Name: `com.protecturownwest`
## App Name: Protect Your Own

---

## Prerequisites

1. **Node.js** (v18 or higher)
2. **Android Studio** (latest version)
3. **Java JDK 17** (required for Android builds)
4. **Android SDK** (installed via Android Studio)

---

## Step 1: Install Dependencies

```bash
npm install
```

---

## Step 2: Build the Web App

```bash
npm run build
```

This creates the `dist` folder with your production build.

---

## Step 3: Initialize Capacitor Android Project

```bash
# Add Android platform
npx cap add android

# Sync the web build to Android
npx cap sync android
```

---

## Step 4: Open in Android Studio

```bash
npx cap open android
```

This opens the Android project in Android Studio.

---

## Step 5: Configure Signing in Android Studio

Since you're handling signing yourself:

1. In Android Studio, go to **Build > Generate Signed Bundle / APK**
2. Select **Android App Bundle**
3. Click **Next**
4. Either:
   - **Create new keystore**: Fill in all details and save securely
   - **Choose existing keystore**: Select your existing .jks file
5. Enter your key alias and passwords
6. Click **Next**

---

## Step 6: Build the AAB

1. Select **release** as the build variant
2. Click **Create**
3. The AAB file will be generated at:
   ```
   android/app/release/app-release.aab
   ```

---

## Alternative: Build via Command Line

If you prefer command line after setting up signing:

### Create `android/keystore.properties`:
```properties
storePassword=YOUR_STORE_PASSWORD
keyPassword=YOUR_KEY_PASSWORD
keyAlias=YOUR_KEY_ALIAS
storeFile=PATH_TO_YOUR_KEYSTORE.jks
```

### Build the AAB:
```bash
cd android
./gradlew bundleRelease
```

The AAB will be at: `android/app/build/outputs/bundle/release/app-release.aab`

---

## Step 7: Upload to Play Store

1. Go to [Google Play Console](https://play.google.com/console)
2. Create a new app or select existing
3. Go to **Release > Production** (or Testing track)
4. Click **Create new release**
5. Upload your `app-release.aab` file
6. Fill in release notes
7. Review and roll out

---

## App Icons & Splash Screen

Before building, you should add your app icons:

### Icon Locations:
```
android/app/src/main/res/mipmap-mdpi/ic_launcher.png      (48x48)
android/app/src/main/res/mipmap-hdpi/ic_launcher.png      (72x72)
android/app/src/main/res/mipmap-xhdpi/ic_launcher.png     (96x96)
android/app/src/main/res/mipmap-xxhdpi/ic_launcher.png    (144x144)
android/app/src/main/res/mipmap-xxxhdpi/ic_launcher.png   (192x192)
```

### Splash Screen:
```
android/app/src/main/res/drawable/splash.png
android/app/src/main/res/drawable-land/splash.png
```

---

## Quick Commands Reference

```bash
# Full build and sync
npm run android:build

# Open Android Studio
npm run android:studio

# Just sync changes
npm run cap:sync

# Copy web assets only
npm run cap:copy
```

---

## Troubleshooting

### "SDK location not found"
Create `android/local.properties`:
```properties
sdk.dir=/Users/YOUR_USERNAME/Library/Android/sdk
```
(Adjust path for your OS)

### Build fails with Java errors
Ensure JAVA_HOME points to JDK 17:
```bash
export JAVA_HOME=/path/to/jdk-17
```

### Capacitor sync issues
```bash
npx cap sync android --force
```

---

## Play Store Requirements Checklist

- [ ] AAB file generated and signed
- [ ] App icons in all required sizes
- [ ] Privacy policy URL
- [ ] App screenshots (phone & tablet)
- [ ] Feature graphic (1024x500)
- [ ] Short description (80 chars max)
- [ ] Full description (4000 chars max)
- [ ] Content rating questionnaire completed
- [ ] Target audience and content declarations
- [ ] Data safety form completed

---

## Support

For Capacitor documentation: https://capacitorjs.com/docs/android
For Play Console help: https://support.google.com/googleplay/android-developer
