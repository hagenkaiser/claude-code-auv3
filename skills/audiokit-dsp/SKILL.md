---
name: audiokit-dsp
description: AudioKit 5 DSP programming for iOS/macOS AUv3 instruments. Use when building samplers, synthesizers, audio effects, or MIDI instruments with AudioKit, SoundpipeAudioKit, or SporthAudioKit.
---

# AudioKit 5 DSP Expert Skill

Expert guidance for audio DSP programming using AudioKit 5 for iOS/macOS AUv3-compatible instruments.

## Critical Rules

1. **AudioKit 5 ONLY** - Never use AudioKit 4 patterns. Use main branch dependencies.
2. **AUv3 First** - All new projects should be AUv3-compatible from the start
3. **Use existing AudioKit nodes** when available before implementing custom DSP
4. **Real-time thread safety** - Never allocate, lock, or block in render callbacks

## Package Dependencies

```swift
dependencies: [
    .package(url: "https://github.com/AudioKit/AudioKit.git", branch: "main"),
    .package(url: "https://github.com/AudioKit/SoundpipeAudioKit.git", branch: "main"),
    .package(url: "https://github.com/AudioKit/SporthAudioKit.git", branch: "main"),
    .package(url: "https://github.com/AudioKit/DunneAudioKit.git", branch: "main"),  // Optional
    .package(url: "https://github.com/AudioKit/Keyboard.git", branch: "main"),
    .package(url: "https://github.com/AudioKit/Tonic.git", from: "1.0.7"),
]
```

## Package Overview

| Package | Contains | Use For |
|---------|----------|---------|
| **AudioKit** | AudioEngine, MIDISampler, Reverb, Delay, MIDI | Core audio, sample playback, basic effects |
| **SoundpipeAudioKit** | DynamicOscillator, MoogLadder, AmplitudeEnvelope | Advanced synthesis, analog filters, envelopes |
| **SporthAudioKit** | OperationGenerator, OperationEffect | Complex DSP graphs, custom generators/effects |
| **DunneAudioKit** | Synth One voices | Advanced polyphonic synthesizers |

## When to Use What

**AudioKit core nodes** - Sample playback (MIDISampler), basic effects, simple synthesis

**SoundpipeAudioKit nodes** - Individual DSP building blocks, standard synthesis patterns, maximum control

**SporthAudioKit Operations** - Complex interconnected DSP, custom generators/effects, dynamic parameter routing

## Real-Time Thread Safety

### NEVER do in render callback:
- Allocate memory (`Array()`, `String()`)
- Use locks (`DispatchSemaphore`, `NSLock`)
- Call Objective-C methods
- Access disk or network
- Update `@Published` properties
- Call `Log()` or `print()`

### SAFE in render callback:
- Read/write pre-allocated buffers
- Simple arithmetic
- Call AudioKit DSP node methods
- Read atomic values

## Core Patterns

### Conductor Pattern

```swift
import AudioKit

class Conductor: ObservableObject {
    let engine = AudioEngine()
    var instrument = MIDISampler(name: "Instrument")
    @Published var reverb: Reverb

    init() {
        reverb = Reverb(instrument)
        reverb.dryWetMix = 0.3
        engine.output = reverb
    }

    func start() {
        do {
            if let url = Bundle.main.url(forResource: "Sounds/Instrument", withExtension: "exs") {
                try instrument.loadInstrument(url: url)
            }
            try engine.start()
        } catch {
            Log("AudioKit failed to start")
        }
    }
}
```

### App Entry Point (iOS)

```swift
import SwiftUI
import AVFoundation
import AudioKit

@main
struct MyApp: App {
    init() {
        #if os(iOS)
        do {
            Settings.bufferLength = .medium
            try AVAudioSession.sharedInstance().setCategory(.playback, options: [.mixWithOthers])
            try AVAudioSession.sharedInstance().setActive(true)
        } catch { print(err) }
        #endif
    }

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

## Additional References

For detailed documentation on specific topics:

- **SporthAudioKit Operations**: See [operations-reference.md](operations-reference.md)
- **AUv3 Implementation**: See [auv3-boilerplate.md](auv3-boilerplate.md)
- **SoundpipeAudioKit Nodes**: See [soundpipe-nodes.md](soundpipe-nodes.md)
