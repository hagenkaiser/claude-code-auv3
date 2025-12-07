# AUv3 Integrator Code Reference

## YourProjectAUv3AudioUnit.swift Structure

```swift
public class YourProjectAUv3AudioUnit: AUAudioUnit {
    // MARK: - Properties
    var engine: AVAudioEngine!
    var conductor: Conductor!
    private var confirmEngineStarted = false
    private var doneLoading = false

    // MARK: - Parameters
    var param1 = AUParameterTree.createParameter(...)

    // MARK: - Init
    public override init(componentDescription:options:) throws

    // MARK: - Parameter Tree
    func setupParamTree()
    func setupParamCallbacks()

    // MARK: - Render Block
    func setInternalRenderingBlock()
    func handleEvents(eventsList:timestamp:)
    func handleMIDI(midiEvent:timestamp:)
    func handleParameter(parameterEvent:timestamp:)

    // MARK: - MIDI
    func receivedMIDINoteOn(noteNumber:velocity:channel:offset:)
    func receivedMIDINoteOff(noteNumber:channel:offset:)

    // MARK: - Lifecycle
    override func allocateRenderResources() throws
    override func deallocateRenderResources()
    func engineStart()

    // MARK: - Presets
    override var factoryPresets: [AUAudioUnitPreset]
    override var currentPreset: AUAudioUnitPreset?

    // MARK: - Bus Configuration
    func setOutputBusArrays() throws
}
```

## AudioUnitViewController.swift

```swift
import CoreAudioKit
import SwiftUI

#if os(iOS)
typealias HostingController = UIHostingController
#elseif os(macOS)
typealias HostingController = NSHostingController
#endif

public class AudioUnitViewController: AUViewController, AUAudioUnitFactory {
    var audioUnit: AUAudioUnit?
    var hostingController: HostingController<YourProjectAUv3View>?
    var parameterObserverToken: AUParameterObserverToken?
    var needsConnection = true

    // Parameter references
    var param1: AUParameter?

    public override func viewDidLoad()
    private func setupParameterObservation()
    public func createAudioUnit(with:) throws -> AUAudioUnit
    private func configureSwiftUIView(audioUnit:)
}
```

## AudioParameter.swift (Bidirectional Binding)

```swift
class AudioParameter: ObservableObject {
    @Published var value: AUValue
    var auParameter: AUParameter

    init(auParameter: AUParameter, initialValue: AUValue) {
        self.auParameter = auParameter
        self.value = initialValue
    }

    func updateValue(_ newValue: AUValue) {
        DispatchQueue.main.async {
            self.value = newValue
            self.auParameter.setValue(newValue, originator: nil)
        }
    }
}
```

## AUParameter Definition

```swift
var filterCutoff = AUParameterTree.createParameter(
    withIdentifier: "filterCutoff",
    name: "Filter Cutoff",
    address: 0,
    min: 20.0,
    max: 20000.0,
    unit: .hertz,
    flags: [.flag_IsReadable, .flag_IsWritable, .flag_CanRamp]
)
```

## Parameter Tree Registration

```swift
parameterTree = AUParameterTree.createTree(withChildren: [filterCutoff, ...])
```

## Value Observer (UI → DSP)

```swift
parameterTree?.implementorValueObserver = { param, value in
    if param.identifier == "filterCutoff" {
        self.conductor.filter.cutoffFrequency = value
    }
}
```

## Parameter Observer Token (DAW → UI)

```swift
parameterObserverToken = paramTree.token(byAddingParameterObserver: { address, value in
    DispatchQueue.main.async {
        self.hostingController?.rootView.updateParameter(address, value: value)
    }
})
```

## MIDI Handling

```swift
private func handleMIDI(midiEvent event: AUMIDIEvent,
                       timestamp: UnsafePointer<AudioTimeStamp>) {
    let midiEvent = MIDIEvent(data: [event.data.0, event.data.1, event.data.2])
    guard let statusType = midiEvent.status?.type else { return }

    switch statusType {
    case .noteOn:
        if midiEvent.data[2] == 0 {
            conductor.noteOff(event.data.1, channel: midiEvent.channel ?? 0)
        } else {
            conductor.noteOn(event.data.1, velocity: event.data.2,
                           channel: midiEvent.channel ?? 0)
        }
    case .noteOff:
        conductor.noteOff(event.data.1, channel: midiEvent.channel ?? 0)
    case .controllerChange:
        handleCC(event.data.1, value: event.data.2, channel: midiEvent.channel ?? 0)
    case .pitchWheel:
        if let amount = midiEvent.pitchbendAmount {
            conductor.setPitchBend(amount, channel: midiEvent.channel ?? 0)
        }
    default:
        break
    }
}
```

## MIDI CC Mapping

```swift
private func handleCC(_ cc: MIDIByte, value: MIDIByte, channel: MIDIChannel) {
    switch cc {
    case 1:  // Mod wheel
        conductor.setModWheel(AUValue(value) / 127.0)
    case 74: // Filter cutoff
        filterCutoff.value = mapCCToRange(value, min: 20, max: 20000)
    case 71: // Resonance
        filterResonance.value = AUValue(value) / 127.0
    default:
        break
    }
}

private func mapCCToRange(_ cc: MIDIByte, min: AUValue, max: AUValue) -> AUValue {
    return min + (max - min) * AUValue(cc) / 127.0
}
```

## Factory Presets

```swift
public override var factoryPresets: [AUAudioUnitPreset] {
    return [
        AUAudioUnitPreset(number: 0, name: "Init"),
        AUAudioUnitPreset(number: 1, name: "Fat Bass"),
        AUAudioUnitPreset(number: 2, name: "Bright Lead"),
    ]
}

public override var currentPreset: AUAudioUnitPreset? {
    get { return _currentPreset }
    set {
        guard let preset = newValue else {
            _currentPreset = nil
            return
        }
        if preset.number >= 0 {
            _currentPreset = preset
            loadFactoryPreset(preset.number)
        }
    }
}

private func loadFactoryPreset(_ number: Int) {
    switch number {
    case 0: // Init
        filterCutoff.value = 1000
        filterResonance.value = 0.5
    case 1: // Fat Bass
        filterCutoff.value = 200
        filterResonance.value = 0.7
    default:
        break
    }
}
```

## Info.plist Configuration

```xml
<key>NSExtension</key>
<dict>
    <key>NSExtensionAttributes</key>
    <dict>
        <key>AudioComponents</key>
        <array>
            <dict>
                <key>type</key>
                <string>aumu</string>
                <key>subtype</key>
                <string>syn1</string>
                <key>manufacturer</key>
                <string>MANU</string>
                <key>name</key>
                <string>MANU: YourSynth</string>
                <key>description</key>
                <string>YourSynthAUv3</string>
                <key>factoryFunction</key>
                <string>$(PRODUCT_MODULE_NAME).AudioUnitViewController</string>
                <key>sandboxSafe</key>
                <true/>
                <key>tags</key>
                <array>
                    <string>Synthesizer</string>
                </array>
                <key>version</key>
                <integer>1</integer>
            </dict>
        </array>
    </dict>
    <key>NSExtensionMainStoryboard</key>
    <string>MainInterface</string>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.AudioUnit-UI</string>
</dict>
```
