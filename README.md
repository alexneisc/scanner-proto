# scanner-proto

A throwaway Tauri 2 + SvelteKit prototype that scans QR codes and barcodes with the iPhone camera. It exists to find out whether the Tauri plugin approach is good enough for a real app, compared with a native implementation. It is not a product and has no license.

What it does: one screen with a Scan button. After a scan it shows the code type (QR, Code 128, EAN-13 and so on), the decoded text, and the time in milliseconds from the moment scanning starts (after the camera permission is granted) to the result. It keeps the last 10 scans in memory. It also has a Flash button that keeps the torch on during a scan. There are no network requests, no database and no analytics.

Tested only on an iPhone 14 Pro with iOS 27. Android was never built or run.

## Versions used

| | |
|---|---|
| Tauri core / CLI | 2.12.5 / 2.12.4 |
| tauri-plugin-barcode-scanner (Rust) | 2.5.3, **patched locally** (see below) |
| @tauri-apps/plugin-barcode-scanner (JS) | 2.5.3 |
| Rust | 1.99.1 |
| Node / npm | 26.12.0 / 12.2.1 |
| Xcode / iOS SDK | 27.2 / 27.2 |

## The plugin patch

`src-tauri/vendor/tauri-plugin-barcode-scanner` is a modified copy of the official plugin. It fixes close-range blur by letting iOS use the ultra-wide lens, and adds a torch command that works during a scan. Read [`src-tauri/vendor/PATCH.md`](src-tauri/vendor/PATCH.md) for the problem, the changes, the status, and the likely similar problem on Android. The upstream plugin's licenses (`LICENSE_APACHE-2.0`, `LICENSE_MIT`) are kept in that directory.

## Build and run on an iPhone

Requirements: Xcode with the iOS SDK and an iOS Simulator runtime, Rust (rustup) with the `aarch64-apple-ios` and `aarch64-apple-ios-sim` targets, Node, CocoaPods, and an Apple Development certificate. A paid developer account is not needed for a personal device.

```sh
npm install
npm run tauri ios init      # generates src-tauri/gen/apple (not committed)
```

Set your Apple Team ID. Either export it:

```sh
export APPLE_DEVELOPMENT_TEAM=YOURTEAMID
```

or create `src-tauri/tauri.ios.conf.json` (it is git-ignored):

```json
{ "bundle": { "iOS": { "developmentTeam": "YOURTEAMID" } } }
```

Then:

```sh
npm run tauri ios dev -- "<your iPhone name>"                   # debug, with dev server
npm run tauri ios build -- --export-method debugging            # release IPA
```

The release IPA is written to `src-tauri/gen/apple/build/arm64/`. Install it with `xcrun devicectl device install app --device <UDID> "<path to .ipa>"`.

On the first run, trust the developer profile on the phone (Settings > General > VPN & Device Management). The camera permission text lives in `src-tauri/Info.ios.plist`.

After `ios init`, `src-tauri/gen/apple/scanner-proto_iOS/Info.plist` is generated from `Info.ios.plist`. If you change the permission text later, check that the generated copy picked it up.

## Capabilities

Only the permissions the scanner needs, for mobile only (`src-tauri/capabilities/mobile.json`): `barcode-scanner:allow-scan`, `allow-cancel`, `allow-request-permissions` and `allow-set-torch` (the last one is added by the local patch).
