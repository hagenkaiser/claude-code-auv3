# AUv3 Implementation Boilerplate

## Project Structure

```
YourProject/
├── YourProject/                          # Main app target
│   ├── YourProjectApp.swift             # App entry point + Settings
│   ├── YourProjectView.swift            # Main SwiftUI view
│   ├── Conductor.swift                  # Audio engine manager
│   ├── ParameterSlider.swift            # Reusable UI component
│   └── SwiftUIKeyboard.swift            # Piano keyboard wrapper
├── YourProjectAUv3/                     # AUv3 extension target
│   ├── Audio Unit/
│   │   └── YourProjectAUv3AudioUnit.swift
│   ├── UI/
│   │   ├── AudioUnitViewController.swift
│   │   └── YourProjectAUv3View.swift
│   └── Info.plist
└── Sounds/                              # Shared audio assets
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
                <string>aumu</string>           <!-- Music Instrument -->
                <key>subtype</key>
                <string>syn1</string>           <!-- 4-char unique ID -->
                <key>manufacturer</key>
                <string>TEST</string>           <!-- 4-char manufacturer -->
                <key>name</key>
                <string>TEST: YourSynth</string>
                <key>factoryFunction</key>
                <string>$(PRODUCT_MODULE_NAME).AudioUnitViewController</string>
                <key>sandboxSafe</key>
                <true/>
                <key>tags</key>
                <array>
                    <string>Synthesizer</string>
                </array>
            </dict>
        </array>
    </dict>
    <key>NSExtensionMainStoryboard</key>
    <string>MainInterface</string>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.AudioUnit-UI</string>
</dict>
```

**Audio Unit Types:**
- `aumu` - Music Instrument (generates sound from MIDI)
- `aufx` - Effect (processes audio input)
- `aumi` - MIDI Processor (transforms MIDI)
- `aumc` - Music Effect (MIDI + audio input)

## Audio Unit Implementation

```swift
import AudioToolbox
import AVFoundation
import CoreAudioKit
import AudioKit

public class YourProjectAUv3AudioUnit: AUAudioUnit {
    var engine: AVAudioEngine!
    var conductor: Conductor!
    private var _currentPreset: AUAudioUnitPreset?
    private var confirmEngineStarted = false
    private var doneLoading = false

    // Define parameters
    var reverbParam = AUParameterTree.createParameter(
        withIdentifier: "reverb", name: "Reverb", address: 0,
        min: 0.0, max: 1.0, unit: .generic, unitName: nil,
        flags: [.flag_IsReadable, .flag_IsWritable, .flag_CanRamp],
        valueStrings: nil, dependentParameters: nil
    )

    public override init(componentDescription: AudioComponentDescription,
                        options: AudioComponentInstantiationOptions = []) throws {
        conductor = Conductor()
        engine = conductor.engine.avEngine

        try super.init(componentDescription: componentDescription, options: options)
        try setOutputBusArrays()

        setupParamTree()
        setupParamCallbacks()
        setInternalRenderingBlock()
    }

    // MARK: - Factory Presets
    public override var factoryPresets: [AUAudioUnitPreset] {
        [AUAudioUnitPreset(number: 0, name: "Dry"),
         AUAudioUnitPreset(number: 1, name: "Wet")]
    }

    // MARK: - Parameter Setup
    public func setupParamTree() {
        reverbParam.value = 0.3
        parameterTree = AUParameterTree.createTree(withChildren: [reverbParam])
    }

    public func setupParamCallbacks() {
        parameterTree?.implementorValueObserver = { param, value in
            if param.identifier == "reverb" {
                self.conductor.reverb.dryWetMix = value
            }
        }
    }

    // MARK: - MIDI Handling
    private func handleMIDI(midiEvent event: AUMIDIEvent,
                           timestamp: UnsafePointer<AudioTimeStamp>) {
        let midiEvent = MIDIEvent(data: [event.data.0, event.data.1, event.data.2])
        guard let statusType = midiEvent.status?.type else { return }

        switch statusType {
        case .noteOn:
            if midiEvent.data[2] == 0 {
                conductor.instrument.stop(noteNumber: event.data.1, channel: midiEvent.channel ?? 0)
            } else {
                conductor.instrument.play(noteNumber: event.data.1, velocity: event.data.2,
                                         channel: midiEvent.channel ?? 0)
            }
        case .noteOff:
            conductor.instrument.stop(noteNumber: event.data.1, channel: midiEvent.channel ?? 0)
        case .controllerChange:
            conductor.instrument.midiCC(event.data.1, value: event.data.2,
                                       channel: midiEvent.channel ?? 0)
        case .pitchWheel:
            if let amount = midiEvent.pitchbendAmount, let ch = midiEvent.channel {
                conductor.instrument.setPitchbend(amount: amount, channel: ch)
            }
        default: break
        }
    }

    // MARK: - Render Block
    private func setInternalRenderingBlock() {
        self._internalRenderBlock = { [weak self] (actionFlags, timestamp, frameCount,
                                                   outputBusNumber, outputData,
                                                   renderEvent, pullInputBlock) in
            guard let self = self else { return 1 }
            if let eventList = renderEvent?.pointee {
                self.handleEvents(eventsList: eventList, timestamp: timestamp)
            }
            _ = self.engine.manualRenderingBlock(frameCount, outputData, nil)
            return noErr
        }
    }

    // MARK: - Lifecycle
    override public func allocateRenderResources() throws {
        try engine.enableManualRenderingMode(.offline, format: outputBus.format,
                                             maximumFrameCount: 4096)
        conductor.start()
        try super.allocateRenderResources()
        doneLoading = true
    }

    override public func deallocateRenderResources() {
        engine.stop()
        super.deallocateRenderResources()
    }

    // MARK: - Bus Setup
    var outputBus: AUAudioUnitBus!
    open var _outputBusArray: AUAudioUnitBusArray!
    override open var outputBusses: AUAudioUnitBusArray { _outputBusArray }

    open func setOutputBusArrays() throws {
        let format = AVAudioFormat(standardFormatWithSampleRate: 44100, channels: 2)!
        outputBus = try AUAudioUnitBus(format: format)
        _outputBusArray = AUAudioUnitBusArray(audioUnit: self, busType: .output, busses: [outputBus])
    }

    // Required properties
    public override var canProcessInPlace: Bool { true }
    open var _parameterTree: AUParameterTree!
    override open var parameterTree: AUParameterTree? {
        get { _parameterTree }
        set { _parameterTree = newValue }
    }
    open var _internalRenderBlock: AUInternalRenderBlock!
    override open var internalRenderBlock: AUInternalRenderBlock { _internalRenderBlock }
}
```

## AudioUnitViewController (SwiftUI Host)

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
    var hostingController: HostingController<YourAUv3View>?
    var reverbParameter: AUParameter?
    var needsConnection = true

    public func createAudioUnit(with componentDescription: AudioComponentDescription) throws -> AUAudioUnit {
        audioUnit = try YourProjectAUv3AudioUnit(componentDescription: componentDescription)
        DispatchQueue.main.async {
            self.setupParameterObservation()
            self.configureSwiftUIView()
        }
        return audioUnit!
    }

    private func setupParameterObservation() {
        guard needsConnection, let paramTree = audioUnit?.parameterTree else { return }
        reverbParameter = paramTree.value(forKey: "reverb") as? AUParameter
        needsConnection = false
    }

    private func configureSwiftUIView() {
        guard let reverbParam = reverbParameter else { return }
        let audioParam = AudioParameter(auParameter: reverbParam, initialValue: reverbParam.value)
        let contentView = YourAUv3View(audioParameter: audioParam)
        let host = HostingController(rootView: contentView)

        addChild(host)
        host.view.frame = view.bounds
        view.addSubview(host.view)
        hostingController = host

        host.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            host.view.topAnchor.constraint(equalTo: view.topAnchor),
            host.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            host.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            host.view.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
    }
}
```

## AUv3 SwiftUI View

```swift
import CoreAudioKit
import SwiftUI

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

struct YourAUv3View: View {
    @ObservedObject var audioParameter: AudioParameter

    var body: some View {
        VStack {
            ParameterSlider(text: "Reverb", parameter: $audioParameter.value,
                           range: 0...1, units: "Percent")
            .onChange(of: audioParameter.value) { newValue in
                audioParameter.updateValue(newValue)
            }
        }
    }
}
```
