# SoundpipeAudioKit DSP Nodes Reference

Individual DSP building blocks for advanced synthesis and audio processing.

## Oscillators

```swift
import SoundpipeAudioKit

// Dynamic Oscillator - switchable waveforms
let osc = DynamicOscillator()
osc.frequency = 440
osc.amplitude = 0.5
osc.setWaveform(Table(.sine))  // .sine, .triangle, .square, .sawtooth

// FM Oscillator
let fm = FMOscillator()
fm.baseFrequency = 440
fm.carrierMultiplier = 1.0
fm.modulatingMultiplier = 1.0
fm.modulationIndex = 1.0

// PWM Oscillator
let pwm = PWMOscillator()
pwm.frequency = 440
pwm.pulseWidth = 0.5

// Morphing Oscillator
let morph = MorphingOscillator()
morph.frequency = 440
morph.index = 0.5
```

## Filters

```swift
// Moog Ladder Filter (classic analog sound)
let moog = MoogLadder(input)
moog.cutoffFrequency = 1000
moog.resonance = 0.5

// Korg 35 Filter (MS-20 style)
let korg = Korg35Filter(input)
korg.cutoffFrequency = 1000
korg.resonance = 0.5
korg.saturation = 0.0

// Roland TB-303 Filter
let tb303 = RolandTB303Filter(input)
tb303.cutoffFrequency = 1000
tb303.resonance = 0.5
```

## Envelopes

```swift
let envelope = AmplitudeEnvelope(oscillator)
envelope.attackDuration = 0.1
envelope.decayDuration = 0.2
envelope.sustainLevel = 0.7
envelope.releaseDuration = 0.5

envelope.start()  // Begin attack
envelope.stop()   // Begin release
```

## Modulation Effects

```swift
// Tremolo
let tremolo = Tremolo(input)
tremolo.frequency = 5.0
tremolo.depth = 0.5

// Vibrato
let vibrato = Vibrato(input)
vibrato.frequency = 5.0
vibrato.depth = 0.5

// Auto Pan
let autoPan = AutoPan(input)
autoPan.frequency = 1.0
autoPan.depth = 1.0
```

## Physical Modeling

```swift
// Plucked String (Karplus-Strong)
let pluck = PluckedString()
pluck.frequency = 110
pluck.amplitude = 0.5

// Clarinet
let clarinet = Clarinet()
clarinet.frequency = 440
clarinet.amplitude = 0.5

// Flute
let flute = Flute()
flute.frequency = 440
flute.amplitude = 0.5
```

## Analysis Nodes

```swift
// Pitch Detection
let pitchTap = PitchTap(input) { frequency, amplitude in
    print("Detected: \(frequency) Hz at \(amplitude)")
}
pitchTap.start()

// Amplitude Tracking
let ampTap = AmplitudeTap(input) { amplitude in
    print("Level: \(amplitude)")
}
ampTap.start()

// FFT Analysis
let fftTap = FFTTap(input) { fftData in
    // Process frequency bins
}
fftTap.start()
```

## Signal Chain Example

```swift
import AudioKit
import SoundpipeAudioKit

class SynthConductor: ObservableObject {
    let engine = AudioEngine()
    var oscillator: DynamicOscillator
    var filter: MoogLadder
    var envelope: AmplitudeEnvelope

    init() {
        oscillator = DynamicOscillator()
        oscillator.setWaveform(Table(.sawtooth))

        filter = MoogLadder(oscillator)
        filter.cutoffFrequency = 2000
        filter.resonance = 0.5

        envelope = AmplitudeEnvelope(filter)
        envelope.attackDuration = 0.01
        envelope.decayDuration = 0.1
        envelope.sustainLevel = 0.7
        envelope.releaseDuration = 0.3

        engine.output = envelope
    }

    func noteOn(frequency: Float) {
        oscillator.frequency = frequency
        oscillator.start()
        envelope.start()
    }

    func noteOff() {
        envelope.stop()
    }

    func start() {
        do { try engine.start() }
        catch { Log("Engine failed") }
    }
}
```
