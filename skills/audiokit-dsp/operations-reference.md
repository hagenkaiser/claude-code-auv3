# SporthAudioKit Operations Reference

Operations allow you to build complex, interconnected DSP using a functional approach. Operations compile to Sporth and run efficiently.

## Creating an OperationGenerator

```swift
import SporthAudioKit

let generator = OperationGenerator { parameters in
    let frequency = parameters[0]
    let amplitude = parameters[1]
    return Operation.sineWave(frequency: frequency, amplitude: amplitude)
}

generator.parameter1 = 440   // frequency
generator.parameter2 = 0.5   // amplitude

engine.output = generator
generator.start()
```

## Creating an OperationEffect

```swift
let effect = OperationEffect(inputNode) { input, parameters in
    let cutoff = parameters[0]
    let resonance = parameters[1]
    return input.moogLadderFilter(cutoffFrequency: cutoff, resonance: resonance)
}

effect.parameter1 = 1000  // cutoff
effect.parameter2 = 0.5   // resonance
engine.output = effect
```

## Stereo Operations

```swift
// Stereo generator
let stereoGen = OperationGenerator(channelCount: 2) { parameters in
    let freq = parameters[0]
    let left = Operation.sineWave(frequency: freq)
    let right = Operation.sineWave(frequency: freq * 1.01)  // Slight detune
    return [left, right]
}

// Stereo effect
let stereoEffect = OperationEffect(input, channelCount: 2) { stereoInput, parameters in
    let left = stereoInput.left.reverberateWithCostello(feedback: 0.6)
    let right = stereoInput.right.reverberateWithCostello(feedback: 0.6)
    return [left, right]
}
```

## Available Oscillators

```swift
// Band-limited (recommended)
Operation.sineWave(frequency: 440, amplitude: 1.0)
Operation.sawtoothWave(frequency: 440, amplitude: 0.5)
Operation.squareWave(frequency: 440, amplitude: 1.0, pulseWidth: 0.5)
Operation.triangleWave(frequency: 440, amplitude: 0.5)

// Non-band-limited (aliasing, but CPU efficient)
Operation.sawtooth(frequency: 440, amplitude: 0.5, phase: 0)
Operation.square(frequency: 440, amplitude: 0.5, phase: 0)
Operation.triangle(frequency: 440, amplitude: 0.5, phase: 0)

// FM Oscillator
Operation.fmOscillator(
    baseFrequency: 440,
    carrierMultiplier: 1.0,
    modulatingMultiplier: 1.0,
    modulationIndex: 1.0,
    amplitude: 0.5
)

// Morphing Oscillator (0=sine, 1=square, 2=saw, 3=reverse saw)
Operation.morphingOscillator(frequency: 440, amplitude: 1.0, index: 0.0)

// Phasor (for table lookup)
Operation.phasor(frequency: 1.0, phase: 0)
```

## Noise Generators

```swift
Operation.whiteNoise(amplitude: 1.0)
Operation.pinkNoise(amplitude: 1.0)
Operation.brownianNoise(amplitude: 1.0)
```

## Physical Modeling

```swift
// Karplus-Strong plucked string
Operation.pluckedString(trigger: Operation.trigger, frequency: 110, amplitude: 0.5, lowestFrequency: 110)

// Vocal tract (CPU intensive!)
Operation.vocalTract(frequency: 160, tonguePosition: 0.5, tongueDiameter: 1.0, tenseness: 0.6, nasality: 0.0)
```

## Filters

```swift
// Moog Ladder (classic analog)
input.moogLadderFilter(cutoffFrequency: 1000, resonance: 0.5)

// Korg 35 (MS-20 style)
input.korgLowPassFilter(cutoffFrequency: 1000, resonance: 1.0, saturation: 0)

// Three-Pole with distortion
input.threePoleLowPassFilter(distortion: 0.5, cutoffFrequency: 1500, resonance: 0.5)

// Butterworth (clean)
input.lowPassButterworthFilter(cutoffFrequency: 1000)
input.highPassButterworthFilter(cutoffFrequency: 500)

// Simple first-order
input.lowPassFilter(halfPowerPoint: 1000)
input.highPassFilter(halfPowerPoint: 1000)

// Resonant
input.resonantFilter(frequency: 4000, bandwidth: 1000)

// Modal resonance (physical modeling)
input.modalResonanceFilter(frequency: 500, qualityFactor: 50)

// String resonator
input.stringResonator(frequency: 100, feedback: 0.95)

// Auto-wah
input.autoWah(wah: 0.5, amplitude: 0.1)

// DC blocking
input.dcBlock()
```

## Delays

```swift
// Basic delay
input.delay(time: 1.0, feedback: 0.5)

// Smooth delay (no pitch shifting when time changes)
input.smoothDelay(time: 1.0, feedback: 0.5, samples: 1024, maximumDelayTime: 5.0)

// Variable delay (cubic interpolation)
input.variableDelay(time: 1.0, feedback: 0.5, maximumDelayTime: 5.0)
```

## Reverbs

```swift
// Chowning (allpass + comb)
input.reverberateWithChowning()

// Comb filter
input.reverberateWithCombFilter(reverbDuration: 1.0, loopDuration: 0.1)

// Costello (stereo FDN) - returns StereoOperation
input.reverberateWithCostello(feedback: 0.6, cutoffFrequency: 4000)

// Flat frequency response
input.reverberateWithFlatFrequencyResponse(reverbDuration: 0.5, loopDuration: 0.1)
```

## Distortion

```swift
input.distort(
    pregain: 2.0,                  // 0-10
    postgain: 0.5,                 // 0-10
    positiveShapeParameter: 0.0,   // -10 to 10
    negativeShapeParameter: 0.0    // -10 to 10
)
```

## Complex Example: FM Synth

```swift
class FMSynthConductor: ObservableObject {
    let engine = AudioEngine()
    let generator: OperationGenerator

    @Published var carrierFrequency: AUValue = 440 {
        didSet { generator.parameter1 = carrierFrequency }
    }
    @Published var modulatorRatio: AUValue = 2.0 {
        didSet { generator.parameter2 = modulatorRatio }
    }
    @Published var modulationIndex: AUValue = 1.0 {
        didSet { generator.parameter3 = modulationIndex }
    }

    init() {
        generator = OperationGenerator { params in
            Operation.fmOscillator(
                baseFrequency: params[0],
                carrierMultiplier: 1.0,
                modulatingMultiplier: params[1],
                modulationIndex: params[2],
                amplitude: 0.5
            )
        }
        generator.parameter1 = carrierFrequency
        generator.parameter2 = modulatorRatio
        generator.parameter3 = modulationIndex

        let reverb = Reverb(generator)
        reverb.dryWetMix = 0.2
        engine.output = reverb
    }

    func start() {
        do {
            try engine.start()
            generator.start()
        } catch { Log("Engine failed") }
    }
}
```
