---
name: sporth-dsp
description: SporthAudioKit for sample-accurate DSP voices in AUv3 instruments. Use when building sample playback, slicers, custom filters, envelopes, or voice pools using Sporth operations.
---

# SporthAudioKit DSP Skill

Expert guidance for SporthAudioKit to build sample-accurate DSP voices in iOS/macOS AUv3 instruments.

## Critical Rules

1. **SporthAudioKit depends on AudioKit and SoundpipeAudioKit** - Include all three packages
2. **Use OperationGenerator for sources, OperationEffect for processors**
3. **Parameters accessed via index** - `parameters[0]`, `parameters[1]`, etc. (up to ~14)
4. **Call generator.start() AFTER engine.start()** - Required for sound output
5. **Trigger parameters need reset** - Set to 1.0 then back to 0.0 after brief delay

## Package Dependencies

```swift
dependencies: [
    .package(url: "https://github.com/AudioKit/AudioKit.git", branch: "main"),
    .package(url: "https://github.com/AudioKit/SoundpipeAudioKit.git", branch: "main"),
    .package(url: "https://github.com/AudioKit/SporthAudioKit.git", branch: "main"),
]
```

## Sample Playback

### Frame-Accurate Playback with phasor + tabread

```sporth
# Load sample, play at original speed
"sample" 0 "path/to/file.wav" loadwav
"sample" 1 sr / phasor tabread
```

**Playback Rate:** `frequency = sampleRate / sampleLength`

### Slice Playback

```sporth
# Play samples 10000-20000 of a 44100-sample file
"sample"
0.2268 0.4535    # start=10000/44100, end=20000/44100
2dup -           # duration (end - start)
1 swap / phasor  # phasor at slice rate
swap + tabread   # offset to start
```

## Filters

| Operation | Description | Parameters |
|-----------|-------------|------------|
| `moogladder` | Moog 4-pole LP (warm) | cutoff, resonance(0-1) |
| `diode` | Diode ladder LP (aggressive) | cutoff, resonance |
| `korg35` | Korg MS-20 style | cutoff, resonance |
| `butlp` / `buthp` | Butterworth LP/HP | cutoff |

## Envelopes

| Operation | Parameters | Best For |
|-----------|------------|----------|
| `adsr` | trigger, atk, dec, sus, rel | Sustained sounds |
| `tenvx` | trigger, atk, hold, rel | Drums, percussive |
| `tenv` | trigger, atk, hold, rel | Plucks |

**Percussive:**
```sporth
0 p 0.001 0.1 0.2 tenvx  # Fast attack, short hold, medium release
```

## Parameter Layout (Sampler Voice)

| Index | Parameter | Range | Default |
|-------|-----------|-------|---------|
| 0 | Trigger | 0/1 | 0 |
| 1 | Playback Rate | 0.25-4.0 | 1.0 |
| 2 | Slice Start | 0.0-1.0 | 0.0 |
| 3 | Slice End | 0.0-1.0 | 1.0 |
| 4 | Filter Cutoff | 20-20000 | 20000 |
| 5 | Filter Resonance | 0.0-1.0 | 0.0 |
| 6-9 | ADSR | varies | varies |
| 10 | Volume | 0.0-1.0 | 1.0 |

## Swift Integration

```swift
import SporthAudioKit

let voice = OperationGenerator { parameters in
    let trigger = parameters[0]
    let pitch = parameters[1]
    let cutoff = parameters[2]

    let env = Operation.trigger(trigger)
        .triggeredEnvelope(attack: 0.001, hold: 0.1, release: 0.3)
    let osc = Operation.sineWave(frequency: pitch)

    return osc.moogLadderFilter(cutoffFrequency: cutoff, resonance: 0.5) * env
}

engine.output = voice
try engine.start()
voice.start()  // CRITICAL: After engine.start()

// Trigger
voice.parameter1 = 1.0
DispatchQueue.main.asyncAfter(deadline: .now() + 0.01) {
    voice.parameter1 = 0.0
}
```

## Debugging

- **No sound?** Check `generator.start()` called after `engine.start()`, trigger reset to 0.0
- **Clicks?** Increase envelope attack (min 0.001s)
- **Wrong pitch?** Verify: `freq = sampleRate / sampleLength`
- **Parameters not working?** Set via `parameter1`, read via `parameters[0]`
