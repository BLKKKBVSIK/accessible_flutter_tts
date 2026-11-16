# accessible_flutter_tts

**Text-to-speech that respects VoiceOver and TalkBack.** A drop-in replacement for
[flutter_tts](https://pub.dev/packages/flutter_tts) 4.2.5: same `FlutterTts` class, same methods,
one import line to change.

![flutter_tts speaks over VoiceOver and ignores its speed. accessible_flutter_tts waits for VoiceOver to finish and uses its speed.](https://raw.githubusercontent.com/BLKKKBVSIK/flutter_tts/main/doc/images/hero.png)

## Why

Blind and low-vision users set their screen reader's speed to what suits them, often much faster
than the default. An app that announces something with flutter_tts gets in their way twice:

- **It talks over the screen reader.** On iOS, the announcement starts while VoiceOver is still
  reading, and the user hears two voices at once.
- **It ignores the user's speed.** The app's speech rate is used, so the announcement is too slow
  for someone used to a fast screen reader, or too fast for someone who slowed it down.

accessible_flutter_tts fixes both, automatically:

| | Screen reader off | VoiceOver or TalkBack on |
|---|---|---|
| **When speech starts** | right away, like flutter_tts | after the screen reader finishes talking |
| **Speech speed** | the rate set by the app, like flutter_tts | the screen reader's speed |

There is nothing to configure. Without a screen reader, the plugin runs flutter_tts's code
unchanged.

## See (and hear) the difference

In this real iPhone recording, VoiceOver reads a text, and the app calls `speak()` half a second
later, while VoiceOver is still talking.

[![Timeline of the recording: flutter_tts talks over VoiceOver for 4.7 seconds; accessible_flutter_tts waits for VoiceOver to finish, then speaks 0.32 seconds later.](https://raw.githubusercontent.com/BLKKKBVSIK/flutter_tts/main/doc/images/overlap_timeline.png)](https://github.com/BLKKKBVSIK/flutter_tts/blob/main/doc/media/overlap_demo.mp4)

▶️ **[Listen to the recording](https://github.com/BLKKKBVSIK/flutter_tts/blob/main/doc/media/overlap_demo.mp4)**
(30 s, sound on).

The speed follows the user's settings too. Same sentence, same rate set by the app (0.5), with the
screen reader at different speeds:

![Sentence duration: flutter_tts always takes 7.2 seconds. accessible_flutter_tts takes 1.7 s with VoiceOver fast, 10.2 s with VoiceOver slow, 2.1 s with TalkBack at 3.69x and 4.6 s with TalkBack at 1.58x.](https://raw.githubusercontent.com/BLKKKBVSIK/flutter_tts/main/doc/images/speed_chart.png)

## Getting started

Add the package:

```sh
flutter pub add accessible_flutter_tts
```

Then change the import:

```dart
// Before
import 'package:flutter_tts/flutter_tts.dart';

// After
import 'package:accessible_flutter_tts/accessible_flutter_tts.dart';
```

That's it. Your existing code keeps working:

```dart
final tts = FlutterTts();

await tts.setLanguage('en-US');
await tts.setSpeechRate(0.5); // Used when no screen reader is running.
await tts.speak('Next stop, Gare de Lyon.');
```

The plugin has its own platform channel and native class names, so it can be installed next to
flutter_tts in the same app, for example while you migrate.

> [!NOTE]
> As with flutter_tts, create `FlutterTts` after Flutter has started: call
> `WidgetsFlutterBinding.ensureInitialized()` first if you create it in `main`.

## How it works

### iOS

- **App in the foreground:** the text is handed to VoiceOver as a queued announcement. VoiceOver
  reads it after its current phrase, with the user's voice and speed.
- **App in the background:** VoiceOver doesn't read announcements there, so the plugin speaks the
  text itself, with VoiceOver's voice and speed
  (`prefersAssistiveTechnologySettings`), without waiting.

### Android (8+)

- A screen reader is detected by **touch exploration**, which TalkBack turns on. Other
  accessibility services, such as Select to Speak, don't change anything.
- `speak()` waits while the screen reader is playing speech, then speaks after 300 ms of silence,
  or after 10 s at most.
- Speech uses the phone's text-to-speech speed, which is the speed TalkBack's own slider sets.

### Known limits

- **iOS:** `stop()` can't take back an announcement already handed to VoiceOver.
- **iOS:** text in another language than the phone's is read at that language's VoiceOver speed,
  not the user's current one. iOS offers no way around it.
- **iOS, background or locked screen:** iOS doesn't tell apps when VoiceOver is talking, so speech
  may overlap VoiceOver there (for example while it reads a notification).

## Tested on

| Platform | Device | Result |
|---|---|---|
| iOS 27.2 | iPhone 17 Pro, VoiceOver on and off | waits for VoiceOver, follows its speed, same as flutter_tts without it |
| Android 14 | Nothing Phone (1), TalkBack on and off | waits for TalkBack, follows its speed, same as flutter_tts without it |

Both Swift Package Manager and CocoaPods are supported on iOS.

**macOS, web and Windows** keep flutter_tts's code unchanged: they work like flutter_tts 4.2.5, with
no screen reader handling.

## Platform setup

### Android

Set the minimum SDK to 24 or higher in `android/app/build.gradle`:

```groovy
minSdkVersion 24
```

Apps targeting Android 11 or later need to declare the text-to-speech service in the `queries` of
`android/app/src/main/AndroidManifest.xml`:

```xml
<queries>
  <intent>
    <action android:name="android.intent.action.TTS_SERVICE" />
  </intent>
</queries>
```

### iOS

iOS 13 or later. No setup needed.

### macOS

macOS 10.15 or later.

### Web

Progress updates only work with the browser's native speech synthesis. Use the default engine to
get them ([Chromium#41195426](https://issues.chromium.org/issues/41195426#comment8)).

## API

The API is the one of flutter_tts 4.2.5.

### Features

| Feature | Android | iOS | macOS | Web | Windows |
|---|:-:|:-:|:-:|:-:|:-:|
| speak, stop | ✅ | ✅ | ✅ | ✅ | ✅ |
| languages, rate, volume, pitch, voices | ✅ | ✅ | ✅ | ✅ | ✅ |
| pause | ✅ (API 26+) | ✅ | | ✅ | ✅ |
| is language available | ✅ | ✅ | ✅ | ✅ | |
| progress (speech marks) | ✅ (API 26+) | ✅ | | ✅ | |
| synthesize to file | ✅ | ✅ | | | |
| engines, silence, queue mode, installed languages | ✅ | | | | |
| shared instance, audio category | | ✅ | | | |
| **waits for the screen reader, uses its speed** | ✅ | ✅ | | | |

### Speaking

```dart
final tts = FlutterTts();

await tts.speak('Hello World');
await tts.stop();
await tts.pause(); // Android (API 26+), iOS, web and Windows. See "Pausing on Android".

List<dynamic> languages = await tts.getLanguages;
await tts.setLanguage('en-US');
await tts.isLanguageAvailable('en-US');

await tts.setSpeechRate(0.5); // Ignored while a screen reader is running.
await tts.setVolume(1.0);
await tts.setPitch(1.0);

// Each voice is a map containing at least `name` and `locale`.
// - Windows (UWP voices) only: gender, identifier
// - iOS, macOS only: quality, gender, identifier
// - Android only: quality, latency, network_required, features
List<Map> voices = await tts.getVoices;
await tts.setVoice({'name': 'Karen', 'locale': 'en-AU'});
await tts.setVoice({'identifier': 'com.apple.voice.compact.en-AU.Karen'}); // iOS, macOS

// Wait for the end of the speech (or of the file synthesis) before returning.
await tts.awaitSpeakCompletion(true);
await tts.awaitSynthCompletion(true);

// iOS, macOS and Android. The last argument is isFullPath (false by default).
await tts.synthesizeToFile('Hello World', Platform.isAndroid ? 'tts.wav' : 'tts.caf', false);
```

### Android only

```dart
await tts.speak('Hello World', focus: true);
await tts.setSilence(2);
await tts.getEngines;
await tts.getDefaultEngine;
await tts.setEngine('com.google.android.tts');
await tts.getDefaultVoice;
await tts.isLanguageInstalled('en-AU');
await tts.areLanguagesInstalled(['en-AU', 'en-US']);
await tts.setQueueMode(1);
await tts.getMaxSpeechInputLength;
await tts.setAudioAttributesForNavigation();
```

#### Pausing on Android

Android's text-to-speech can't pause, so the plugin works around it: it remembers where the speech
was (from `onRangeStart()`, API 26+) and starts from there on the next `speak` call. If you use the
`start` and `end` offsets of `setProgressHandler()` with `pause`, keep track of them: they restart
from the new text after a pause.

### iOS only

Use the shared audio
[instance](https://developer.apple.com/documentation/avfoundation/avaudiosession/1616504-sharedinstance):

```dart
await tts.setSharedInstance(true);
```

Set the audio [category and options](https://developer.apple.com/documentation/avfoundation/avaudiosession),
with an optional [mode](https://developer.apple.com/documentation/avfaudio/avaudiosession/mode).
This setup lets background music keep playing during the speech:

```dart
await tts.setIosAudioCategory(
  IosTextToSpeechAudioCategory.ambient,
  [
    IosTextToSpeechAudioCategoryOptions.allowBluetooth,
    IosTextToSpeechAudioCategoryOptions.allowBluetoothA2DP,
    IosTextToSpeechAudioCategoryOptions.mixWithOthers,
  ],
  IosTextToSpeechAudioMode.voicePrompt,
);
```

### Callbacks

```dart
tts.setStartHandler(() => setState(() => ttsState = TtsState.playing));
tts.setCompletionHandler(() => setState(() => ttsState = TtsState.stopped));
tts.setCancelHandler(() => setState(() => ttsState = TtsState.stopped));
tts.setErrorHandler((message) => setState(() => ttsState = TtsState.stopped));

tts.setProgressHandler((String text, int start, int end, String word) {
  setState(() => currentWord = word);
});

// Android, iOS and web
tts.setPauseHandler(() => setState(() => ttsState = TtsState.paused));
tts.setContinueHandler(() => setState(() => ttsState = TtsState.continued));
```

## Example

The [example app](https://github.com/BLKKKBVSIK/flutter_tts/tree/main/example) puts flutter_tts and
accessible_flutter_tts side by side. Turn VoiceOver or TalkBack on and compare.

## Author

Made by **Enzo Conty** ([contact@enzoconty.dev](mailto:contact@enzoconty.dev)) at Ezymob.

accessible_flutter_tts is a fork of [flutter_tts](https://github.com/dlutton/flutter_tts) by
Daniel Lutton and its contributors. MIT license.
