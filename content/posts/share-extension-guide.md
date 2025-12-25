---
draft: false
title: "Share Extension Guide"
date: 2025-12-25
description: "iOS Share Extension Implementation Guide"
---

# iOS Share Extension Implementation Guide

**What This Guide Teaches:** How to add a share extension to your iOS Capacitor app so users can share content from other apps (like YouTube) directly into your app.

**Result:** Users tap "Share" in YouTube → Select your app → Video appears in your app ready to save.

---

## Table of Contents

1. [What is a Share Extension?](#what-is-a-share-extension)
2. [Step-by-Step Implementation](#step-by-step-implementation)
3. [Testing Your Share Extension](#testing-your-share-extension)
4. [Simple Test App](#simple-test-app)
5. [Lessons Learned](#lessons-learned)
6. [Troubleshooting](#troubleshooting)

---

## What is a Share Extension?

A **Share Extension** is a mini-app that appears in iOS's share sheet (the popup when you tap the share button). It allows users to send content from one app to another.

**Example Flow:**

1. User watches a YouTube video
2. Taps the Share button
3. Sees "TubeStreak" in the share sheet
4. Taps it → Extension opens
5. Optionally enters a name
6. Taps "Post" → Main app opens with the video URL

**Key Concept:** The share extension is separate from your main app. They communicate using:

- **URL Schemes** - Like `tubestreak://share?url=...` (a custom URL that opens your app)
- **App Groups** - Shared storage both apps can access (optional backup)

---

## Step-by-Step Implementation

### Phase 1: Create the Share Extension in Xcode

#### Step 1.1: Open Your Project in Xcode

```bash
# From your project root
cd ios/App
open App.xcworkspace
```

**Important:** Always open the `.xcworkspace` file, NOT `.xcodeproj`!

#### Step 1.2: Add New Target (Share Extension)

1. In Xcode, click on your project name in the left sidebar (the blue icon at the top)
2. At the bottom of the target list, click the **+** button (or File → New → Target)
3. Search for "Share Extension"
4. Select **"Share Extension"** and click **Next**
5. Fill in the details:
   - **Product Name:** `TubeStreakShare` (or `YourAppNameShare`)
   - **Language:** Swift
   - **Project:** App
   - **Embed in Application:** App
6. Click **Finish**
7. When asked "Activate 'TubeStreakShare' scheme?", click **Cancel** (we'll run the main app, not the extension)

**What Happened:** Xcode created a new folder `TubeStreakShare` with these files:

- `ShareViewController.swift` - The extension's code
- `Info.plist` - Extension settings
- `MainInterface.storyboard` - UI (we won't use this)

#### Step 1.3: Configure What Content Your Extension Accepts

1. In the left sidebar, expand `TubeStreakShare` folder
2. Click on `Info.plist`
3. Find `NSExtension` → `NSExtensionAttributes` → `NSExtensionActivationRule`
4. Right-click on it and select **Show Raw Keys/Values**
5. Replace the entire `NSExtensionActivationRule` dictionary with this:

```xml
<key>NSExtensionActivationRule</key>
<dict>
    <key>NSExtensionActivationSupportsText</key>
    <true/>
    <key>NSExtensionActivationSupportsWebURLWithMaxCount</key>
    <integer>1</integer>
</dict>
```

**What This Means:**

- Your extension accepts text (YouTube shares as plain text)
- Your extension accepts 1 web URL
- This makes your extension appear when sharing YouTube videos, links, etc.

#### Step 1.4: Create App Group (For Communication Between Extension and Main App)

**What is an App Group?** A shared storage space both your main app and extension can access.

1. Select your main **App** target (not the share extension)
2. Click on **Signing & Capabilities** tab
3. Click **+ Capability** button
4. Search for "App Groups" and double-click it
5. Click the **+** button under App Groups
6. Enter: `group.com.yourcompany.yourappname` (e.g., `group.com.buggasoftware.tubestreak`)
7. Click **OK**

8. Now select your **TubeStreakShare** target (the extension)
9. Repeat steps 2-7 with the **SAME** app group name

**Critical:** Both targets must use the EXACT same group name!

#### Step 1.5: Add URL Scheme to Main App

**What is a URL Scheme?** A custom URL like `tubestreak://` that opens your app (like how `http://` opens Safari).

1. Select your main **App** target
2. Click on **Info** tab
3. Expand **URL Types** section (scroll down if needed)
4. If it's empty, click the **+** button to add one
5. Fill in:
   - **Identifier:** `$(PRODUCT_BUNDLE_IDENTIFIER)`
   - **URL Schemes:** `tubestreak` (lowercase, no `://`)
   - **Role:** Editor

**What This Does:** Now when you open `tubestreak://anything`, it opens your app!

#### Step 1.6: Write the Share Extension Code

1. Open `TubeStreakShare/ShareViewController.swift`
2. **DELETE ALL** existing code
3. Paste this code:

```swift
import UIKit
import Social
import MobileCoreServices
import UniformTypeIdentifiers

class ShareViewController: SLComposeServiceViewController {

    override func isContentValid() -> Bool {
        // Require at least some text (routine name)
        return contentText.count > 0
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        NSLog("🔥 SHARE EXTENSION LOADED")
        placeholder = "Enter a name (optional)"
    }

    override func didSelectPost() {
        NSLog("🔥 didSelectPost CALLED")

        // Get what user typed
        var routineName: String? = self.contentText
        NSLog("📝 User entered: \(String(describing: routineName))")

        // Extract the shared URL
        if let item = extensionContext?.inputItems.first as? NSExtensionItem,
           let attachments = item.attachments {

            NSLog("📎 Attachments count: \(attachments.count)")

            for (index, attachment) in attachments.enumerated() {
                NSLog("📎 Attachment \(index): \(attachment)")

                // Try public.url first
                if attachment.hasItemConformingToTypeIdentifier("public.url") {
                    NSLog("✅ Found public.url attachment")
                    attachment.loadItem(forTypeIdentifier: "public.url", options: nil) { [weak self] (url, error) in
                        let finalName = (url as? URL)?.absoluteString == routineName ? nil : routineName
                        self?.handleURL(url: url as? URL, error: error, routineName: finalName)
                    }
                    return
                }
                // YouTube shares as plain text
                else if attachment.hasItemConformingToTypeIdentifier("public.plain-text") {
                    NSLog("✅ Found public.plain-text attachment (YouTube format)")
                    attachment.loadItem(forTypeIdentifier: "public.plain-text", options: nil) { [weak self] (text, error) in
                        guard let self = self else { return }

                        if let urlString = text as? String, let url = URL(string: urlString) {
                            NSLog("✅ Converted plain text to URL: \(urlString)")
                            let finalName = urlString == routineName ? nil : routineName
                            self.handleURL(url: url, error: nil, routineName: finalName)
                        } else {
                            NSLog("❌ Could not convert plain text to URL")
                            self.extensionContext?.completeRequest(returningItems: [], completionHandler: nil)
                        }
                    }
                    return
                }
            }
        }

        // If no URL found, just close
        NSLog("❌ No URL found - closing extension")
        self.extensionContext?.completeRequest(returningItems: [], completionHandler: nil)
    }

    override func configurationItems() -> [Any]! {
        return []
    }

    // Helper function to handle URL from any source
    private func handleURL(url: URL?, error: Error?, routineName: String?) {
        NSLog("🔍 handleURL - url: \(String(describing: url)), error: \(String(describing: error)), routineName: \(String(describing: routineName))")

        guard let shareURL = url else {
            NSLog("❌ No URL found in handleURL")
            self.extensionContext?.completeRequest(returningItems: [], completionHandler: nil)
            return
        }

        // Build custom URL to open main app
        let youtubeURL = shareURL.absoluteString
        let encodedURL = youtubeURL.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
        let encodedName = routineName?.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""

        var urlString = "tubestreak://share?url=\(encodedURL)"
        if !encodedName.isEmpty {
            urlString += "&name=\(encodedName)"
        }
        let appURL = URL(string: urlString)!

        NSLog("🔥 OPENING MAIN APP WITH: \(appURL.absoluteString)")

        // Save to App Group as backup
        if let sharedDefaults = UserDefaults(suiteName: "group.com.buggasoftware.tubestreak") {
            sharedDefaults.set(appURL.absoluteString, forKey: "sharedURL")
            sharedDefaults.set(Date().timeIntervalSince1970, forKey: "sharedTimestamp")
            sharedDefaults.synchronize()
            NSLog("✅ Saved to App Group (backup)")
        }

        // Open main app using URL scheme
        DispatchQueue.main.async {
            self.openURLInMainApp(appURL)
        }
    }

    @objc func openURLInMainApp(_ url: URL) {
        NSLog("📱 Opening main app with URL")

        // Try to find UIApplication in responder chain
        var responder: UIResponder? = self
        while responder != nil {
            if let application = responder as? UIApplication {
                NSLog("🚀 Found UIApplication, opening URL...")
                application.open(url, options: [:]) { success in
                    NSLog("📱 URL open result: \(success)")
                }
                break
            }
            responder = responder?.next
        }

        // If UIApplication not found, try selector approach
        if responder == nil {
            NSLog("🔄 UIApplication not found, trying selector approach...")
            let selector = NSSelectorFromString("openURL:")
            var responder: UIResponder? = self
            while responder != nil {
                if responder?.responds(to: selector) == true {
                    NSLog("🚀 Found responder that can open URL")
                    responder?.perform(selector, with: url)
                    break
                }
                responder = responder?.next
            }
        }

        // Close the extension after a delay
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) { [weak self] in
            NSLog("📱 Closing share extension")
            self?.extensionContext?.completeRequest(returningItems: [], completionHandler: nil)
        }
    }
}
```

**What This Code Does:**

1. Shows a popup when user shares a URL
2. Lets user optionally enter a name
3. Extracts the shared URL (handles both `public.url` and `public.plain-text` for YouTube)
4. Builds a custom URL like `tubestreak://share?url=YOUTUBE_URL&name=NAME`
5. Opens your main app with that URL
6. Closes the extension

**Important:** Change `group.com.buggasoftware.tubestreak` to YOUR app group name (line 93)!

---

### Phase 2: Handle Incoming URLs in Your React App

#### Step 2.1: Install Required Package

```bash
npm install @capacitor/app
```

#### Step 2.2: Create a Hook to Listen for URLs

Create `src/hooks/useShareHandler.ts`:

```typescript
import { useEffect, useRef } from "react";
import { App as CapacitorApp, URLOpenListenerEvent } from "@capacitor/app";
import { useNavigate } from "react-router-dom";

export const useShareHandler = (
  onShare: (url: string, name?: string) => void
) => {
  const navigate = useNavigate();
  const onShareRef = useRef(onShare);

  // Keep ref up to date
  useEffect(() => {
    onShareRef.current = onShare;
  }, [onShare]);

  useEffect(() => {
    let listenerHandle: { remove: () => void } | null = null;
    let hasProcessedLaunchUrl = false;

    const setupListener = async () => {
      // Listen for app being opened with a URL (when app is already running)
      listenerHandle = await CapacitorApp.addListener(
        "appUrlOpen",
        (event: URLOpenListenerEvent) => {
          console.log("🔗 App opened with URL:", event.url);

          try {
            const urlObj = new URL(event.url);

            // Check if it's our custom scheme: tubestreak://share?url=...
            if (urlObj.hostname === "share") {
              const sharedUrl = urlObj.searchParams.get("url");
              const customName = urlObj.searchParams.get("name");

              if (sharedUrl) {
                console.log("✅ Received shared URL:", sharedUrl, customName);
                onShareRef.current(sharedUrl, customName || undefined);
              }
            }
          } catch (error) {
            console.error("❌ Error handling URL:", error);
          }
        }
      );
    };

    setupListener();

    // Handle cold start - when app was completely closed and opened via share
    if (!hasProcessedLaunchUrl) {
      hasProcessedLaunchUrl = true;
      CapacitorApp.getLaunchUrl().then((launchUrl) => {
        if (launchUrl?.url) {
          console.log("🚀 App launched with URL:", launchUrl.url);
          try {
            const urlObj = new URL(launchUrl.url);
            if (urlObj.hostname === "share") {
              const sharedUrl = urlObj.searchParams.get("url");
              const customName = urlObj.searchParams.get("name");
              if (sharedUrl) {
                onShareRef.current(sharedUrl, customName || undefined);
              }
            }
          } catch (error) {
            console.error("❌ Error handling launch URL:", error);
          }
        }
      });
    }

    return () => {
      if (listenerHandle) {
        listenerHandle.remove();
      }
    };
  }, [navigate]);
};
```

**What This Hook Does:**

1. Listens for your app being opened with a custom URL
2. Parses the URL to extract `url` and `name` parameters
3. Calls the `onShare` callback with the data
4. Handles both cases: app already running OR app was closed

#### Step 2.3: Use the Hook in Your App

In `src/App.tsx`, add:

```typescript
import { useShareHandler } from "./hooks/useShareHandler";

function App() {
  const [sharedData, setSharedData] = useState<{
    url: string;
    name?: string;
  } | null>(null);

  // Listen for shared URLs
  useShareHandler((url, name) => {
    console.log("📱 Received share:", url, name);
    setSharedData({ url, name });
    // Do whatever you want with the shared URL!
    // For example: navigate to a page, show a modal, save to database, etc.
  });

  // Rest of your app...
}
```

**What This Does:**

- When user shares from YouTube → Your callback runs with the YouTube URL
- You can save it, show a modal, navigate, etc.

---

### Phase 3: Build and Test

#### Step 3.1: Build Your React App

```bash
npm run build
npx cap sync ios
```

**What This Does:**

- Builds your React app to production
- Copies files to the iOS project
- Updates Capacitor plugins

#### Step 3.2: Build in Xcode

1. In Xcode, select your **physical iPhone** from the device dropdown (top toolbar)
   - Share extensions don't work in the iOS Simulator!
2. Press **⌘K** (Product → Clean Build Folder)
3. Press **⌘B** (Product → Build)
4. Fix any errors if they appear
5. Press **⌘R** (Product → Run) to install on your iPhone

#### Step 3.3: Test the Share Extension

1. Open the **YouTube app** on your iPhone
2. Play any video
3. Tap the **Share button** (arrow pointing up)
4. Scroll through the share sheet and look for your app name
   - If you don't see it, tap **"More"** and enable it
5. Tap your app icon
6. Your share extension should open with a text field
7. Optionally enter a name and tap **"Post"**
8. Your main app should open!
9. Check Xcode console for logs starting with 🔥, ✅, 📱

**Expected Logs:**

```
🔥 SHARE EXTENSION LOADED
🔥 didSelectPost CALLED
📝 User entered: Morning Workout
✅ Found public.plain-text attachment (YouTube format)
✅ Converted plain text to URL: https://youtube.com/watch?v=...
🔥 OPENING MAIN APP WITH: tubestreak://share?url=https%3A%2F%2F...
🚀 Found UIApplication, opening URL...
📱 URL open result: true
📱 Closing share extension

// Then in your React app:
🚀 App launched with URL: tubestreak://share?url=...
✅ Received shared URL: https://youtube.com/watch?v=... Morning Workout
```

---

## Simple Test App

Here's a minimal Capacitor app to test share extensions (useful for learning):

### Create Test App

```bash
# Create new Capacitor app
npm init @capacitor/app

# Choose:
# - Name: ShareTest
# - Package ID: com.yourname.sharetest
# - Framework: React

cd sharetest
npm install
npm install @capacitor/app

# Add iOS platform
npx cap add ios
```

### Minimal Test Code

**src/App.tsx:**

```typescript
import { useState } from "react";
import { App as CapacitorApp } from "@capacitor/app";

function App() {
  const [sharedUrl, setSharedUrl] = useState("");

  // Listen for shared URLs
  useState(() => {
    CapacitorApp.addListener("appUrlOpen", (event) => {
      console.log("Received URL:", event.url);

      try {
        const url = new URL(event.url);
        if (url.hostname === "share") {
          const shared = url.searchParams.get("url");
          if (shared) {
            setSharedUrl(shared);
          }
        }
      } catch (e) {
        console.error("Error:", e);
      }
    });
  }, []);

  return (
    <div style={{ padding: 20 }}>
      <h1>Share Extension Test</h1>
      {sharedUrl && (
        <div>
          <h2>Received Share:</h2>
          <p>{sharedUrl}</p>
        </div>
      )}
    </div>
  );
}

export default App;
```

Then follow steps 1.2-1.6 to add the share extension, and test!

---

## Lessons Learned

### 1. **Share Extensions are Separate Apps**

**Problem:** I thought the share extension code runs inside the main app.

**Reality:** The share extension is a completely separate mini-app with its own lifecycle. It has:

- Its own `Info.plist`
- Its own bundle identifier (like `com.app.main.ShareExtension`)
- Its own memory space
- Limited capabilities (can't directly access main app's data)

**Why This Matters:** You need communication mechanisms (URL schemes, App Groups) for them to talk.

---

### 2. **Share Extensions Can't Directly Open URLs**

**Problem:** Initially tried `extensionContext.open(url)` to open the main app.

**Reality:** Share extensions have restricted permissions. `NSExtensionContext.open()` works for **Action Extensions**, but NOT **Share Extensions**.

**Solution:** Use the responder chain to find `UIApplication` and call its `open()` method:

```swift
var responder: UIResponder? = self
while responder != nil {
    if let application = responder as? UIApplication {
        application.open(url, options: [:]) { success in
            print("Opened: \(success)")
        }
        break
    }
    responder = responder?.next
}
```

**Lesson:** Share extensions have limited permissions - work within those constraints.

---

### 3. **YouTube Shares as Plain Text, Not URL**

**Problem:** Only checked for `public.url` type, YouTube shares weren't detected.

**Reality:** When sharing a YouTube video, iOS passes it as `public.plain-text` with the URL as a string.

**Solution:** Check for BOTH types:

```swift
if attachment.hasItemConformingToTypeIdentifier("public.url") {
    // Handle URL type
} else if attachment.hasItemConformingToTypeIdentifier("public.plain-text") {
    // Handle plain text (YouTube)
    attachment.loadItem(forTypeIdentifier: "public.plain-text", options: nil) { (text, error) in
        if let urlString = text as? String, let url = URL(string: urlString) {
            // Process the URL
        }
    }
}
```

**Lesson:** Different apps share content in different formats - handle multiple types.

---

### 4. **URL Encoding is Critical**

**Problem:** Passed URLs with special characters (`&`, `?`, `=`) unencoded, breaking the URL scheme.

**Reality:** URL query parameters MUST be encoded. A URL like:

```
tubestreak://share?url=https://youtube.com/watch?v=ABC&si=XYZ
```

Gets parsed as:

- `url` = `https://youtube.com/watch?v=ABC`
- `si` = `XYZ` (separate parameter!)

**Solution:** Use `.urlQueryAllowed` character set:

```swift
let encodedURL = youtubeURL.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
let urlString = "tubestreak://share?url=\(encodedURL)"
```

Result:

```
tubestreak://share?url=https%3A%2F%2Fyoutube.com%2Fwatch%3Fv%3DABC%26si%3DXYZ
```

**Lesson:** Always encode data in URL parameters to preserve special characters.

---

### 5. **React Infinite Re-render Loop with useEffect**

**Problem:** The app called `getLaunchUrl()` hundreds of times in a loop.

**Logs:**

```
🚀 App launched with URL: tubestreak://share?url=...
🚀 App launched with URL: tubestreak://share?url=...
🚀 App launched with URL: tubestreak://share?url=...
// (repeating forever)
```

**Root Cause:** The `useEffect` dependency array included `onShare`:

```typescript
useEffect(() => {
  CapacitorApp.getLaunchUrl().then((launchUrl) => {
    if (launchUrl?.url) {
      onShare(url, name); // Triggers re-render
    }
  });
}, [onShare]); // ❌ onShare changes every render!
```

Since `onShare` is recreated on every render, the effect runs again, which calls `onShare`, which triggers a render, creating an infinite loop.

**Solution:** Use `useRef` to store the callback:

```typescript
const onShareRef = useRef(onShare);

useEffect(() => {
  onShareRef.current = onShare;
}, [onShare]);

useEffect(() => {
  let hasProcessedLaunchUrl = false;

  if (!hasProcessedLaunchUrl) {
    hasProcessedLaunchUrl = true;
    CapacitorApp.getLaunchUrl().then((launchUrl) => {
      if (launchUrl?.url) {
        onShareRef.current(url, name); // ✅ Uses ref
      }
    });
  }
}, []); // ✅ No dependencies = runs once
```

**Lesson:** Be careful with `useEffect` dependencies. Use `useRef` for callbacks to avoid infinite loops.

---

### 6. **Capacitor Plugins Require Bridging Headers (If Using Swift + Obj-C)**

**Problem:** Created a custom Capacitor plugin but got `{"code":"UNIMPLEMENTED"}` error.

**Reality:** Capacitor uses Objective-C macros (`CAP_PLUGIN`) to register plugins. If your plugin is in Swift, you need a **bridging header** to expose Objective-C to Swift.

**Solution:** Create `ios/App/App/App-Bridging-Header.h`:

```objc
#import <Capacitor/Capacitor.h>
```

And set it in Xcode build settings:

- Target → Build Settings → Swift Compiler - General
- "Objective-C Bridging Header" = `App/App-Bridging-Header.h`

**Lesson:** We avoided this by using URL schemes instead of a custom plugin!

---

### 7. **Always Test on a Real Device**

**Problem:** Share extensions don't appear in the iOS Simulator.

**Reality:** The iOS Simulator doesn't fully support inter-app communication features like share extensions.

**Solution:** Always test share extensions on a **physical iPhone/iPad**.

**Lesson:** Some iOS features require real hardware - keep a test device handy.

---

### 8. **App Groups are Optional but Useful**

**Discovery:** You can use EITHER URL schemes OR App Groups to pass data.

**We Used:** URL schemes as the primary method, App Groups as backup.

**Why URL Schemes Won:**

- Simpler to implement
- Automatically opens the main app
- Works reliably
- No custom Capacitor plugin needed

**When to Use App Groups:**

- Passing large amounts of data (URLs in our case are small)
- Background data sync
- Widget extensions
- Today extensions

**Lesson:** Choose the simplest solution that works. URL schemes are perfect for share extensions.

---

### 9. **NSLog is More Reliable Than print() in Extensions**

**Problem:** Sometimes `print()` statements didn't appear in Xcode console.

**Reality:** Share extensions run in a restricted environment. `NSLog()` is more reliable for debugging.

**Solution:** Use `NSLog()` for all extension logging:

```swift
NSLog("🔥 Extension loaded")  // ✅ Appears in console
print("Extension loaded")      // ❌ Might not appear
```

**Lesson:** Use `NSLog()` in extensions for reliable debugging.

---

### 10. **User Must Enable Your Extension First Time**

**Discovery:** After installing the app, the extension doesn't appear in share sheet automatically.

**Reality:** Users must manually enable your extension the first time:

1. Tap Share button
2. Scroll to the end of share sheet
3. Tap "More" or "Edit Actions"
4. Toggle your extension ON

**After enabling once**, it always appears.

**Lesson:** Include instructions in your app for first-time users to enable the share extension.

---

## Troubleshooting

### Extension Doesn't Appear in Share Sheet

**Check:**

1. Are you testing on a **real device**? (Not simulator)
2. Did you configure `NSExtensionActivationRule` in `Info.plist`?
3. Is the extension enabled in share sheet settings?
   - Share something → Tap "More" → Enable your extension
4. Did you rebuild after adding the extension?

### Extension Opens But Main App Doesn't Open

**Check:**

1. Did you add the URL scheme to main app's `Info.plist`?
   - Target → Info → URL Types → Add `tubestreak`
2. Check Xcode console for "URL open result: false"
   - Means the URL scheme isn't registered
3. Try a different URL scheme name (avoid common words)

### No Logs Appearing

**Check:**

1. Xcode console filter - clear any search filters
2. Select correct device in console dropdown (top right)
3. Use `NSLog()` instead of `print()`
4. Make sure you're looking at the right target's logs

### "App launched with URL" Repeating Infinitely

**Cause:** Infinite re-render loop in React

**Fix:**

1. Check `useEffect` dependencies
2. Use `useRef` for callbacks
3. Add `hasProcessedLaunchUrl` flag to only check launch URL once

### Build Errors About Missing Bridging Header

**Cause:** Xcode is looking for a bridging header that doesn't exist

**Fix:**

1. Target → Build Settings → Search "bridging"
2. Delete the path in "Objective-C Bridging Header"
3. Clean build folder (⌘K)
4. Rebuild

### Extension Crashes When Tapping "Post"

**Check:**

1. Look at crash logs in Xcode console
2. Common causes:
   - Force-unwrapping `nil` values (using `!`)
   - Array index out of bounds
   - App Group name mismatch

**Fix:** Add `guard let` and `if let` checks:

```swift
guard let url = url else {
    NSLog("❌ No URL")
    return
}
```

---

## Summary

**What You Built:**

1. ✅ Share extension that captures YouTube URLs
2. ✅ URL scheme to open your main app
3. ✅ React hook to handle incoming URLs
4. ✅ Complete data flow from YouTube → Your app

**Key Technologies:**

- **Share Extension** - iOS feature for inter-app sharing
- **URL Schemes** - Custom URLs to open your app
- **Capacitor App API** - Listen for URL events in React
- **App Groups** - Optional shared storage

**Time to Implement:** 1-2 hours for first time, 30 minutes after you've done it once

**Complexity:** Moderate (requires both iOS/Swift and React knowledge)

**Best Use Cases:**

- Saving content from other apps
- Quick capture workflows
- Social media clients
- Read-later apps
- Bookmark managers

---

## Next Steps

1. **Customize the UI:** Edit `ShareViewController.swift` to add custom fields
2. **Support More Content Types:** Add support for images, PDFs, etc.
3. **Add Validation:** Check if URL is valid before opening main app
4. **Error Handling:** Show alerts if something goes wrong
5. **Analytics:** Track what content users are sharing

**Resources:**

- [Apple Docs: Share Extension](https://developer.apple.com/documentation/uikit/share_extensions)
- [Capacitor App API](https://capacitorjs.com/docs/apis/app)
- [URL Schemes](https://developer.apple.com/documentation/xcode/defining-a-custom-url-scheme-for-your-app)

---
