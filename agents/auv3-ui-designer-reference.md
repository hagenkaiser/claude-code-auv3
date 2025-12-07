# AUv3 UI Designer Component Reference

## Knob Control

```swift
struct Knob: View {
    @Binding var value: AUValue
    var range: ClosedRange<AUValue>
    var label: String
    var unit: String = ""
    var accentColor: Color = .orange

    @State private var lastAngle: Double = 0
    @GestureState private var isDragging = false

    private let minAngle: Double = -135
    private let maxAngle: Double = 135

    var body: some View {
        VStack(spacing: 8) {
            ZStack {
                // Background ring
                Circle()
                    .stroke(Color.gray.opacity(0.3), lineWidth: 4)

                // Value arc
                Circle()
                    .trim(from: 0, to: normalizedValue)
                    .stroke(accentColor, style: StrokeStyle(lineWidth: 4, lineCap: .round))
                    .rotationEffect(.degrees(-90))

                // Knob body
                Circle()
                    .fill(
                        LinearGradient(
                            colors: [Color.gray.opacity(0.4), Color.gray.opacity(0.2)],
                            startPoint: .top,
                            endPoint: .bottom
                        )
                    )
                    .padding(6)

                // Indicator line
                Rectangle()
                    .fill(accentColor)
                    .frame(width: 3, height: 15)
                    .offset(y: -20)
                    .rotationEffect(.degrees(angleForValue))
            }
            .frame(width: 60, height: 60)
            .gesture(dragGesture)

            // Label
            Text(label)
                .font(.caption2)
                .foregroundColor(.gray)

            // Value display
            Text(formattedValue)
                .font(.caption)
                .monospacedDigit()
        }
    }

    // ... gesture and calculation logic
}
```

## Vertical Slider (Fader Style)

```swift
struct VerticalSlider: View {
    @Binding var value: AUValue
    var range: ClosedRange<AUValue>
    var label: String
    var accentColor: Color = .blue

    var body: some View {
        VStack(spacing: 4) {
            GeometryReader { geometry in
                ZStack(alignment: .bottom) {
                    // Track background
                    RoundedRectangle(cornerRadius: 4)
                        .fill(Color.black.opacity(0.3))
                        .frame(width: 8)

                    // Filled portion
                    RoundedRectangle(cornerRadius: 4)
                        .fill(accentColor)
                        .frame(width: 8, height: geometry.size.height * CGFloat(normalizedValue))

                    // Fader cap
                    RoundedRectangle(cornerRadius: 4)
                        .fill(Color.gray)
                        .frame(width: 24, height: 12)
                        .offset(y: -geometry.size.height * CGFloat(normalizedValue) + 6)
                }
                .frame(maxWidth: .infinity)
                .gesture(/* drag gesture */)
            }

            Text(label)
                .font(.caption2)
                .foregroundColor(.gray)
        }
    }
}
```

## ADSR Envelope Display

```swift
struct ADSRView: View {
    @Binding var attack: AUValue
    @Binding var decay: AUValue
    @Binding var sustain: AUValue
    @Binding var release: AUValue

    var body: some View {
        VStack {
            // Visual envelope curve
            GeometryReader { geometry in
                Path { path in
                    let w = geometry.size.width
                    let h = geometry.size.height

                    let attackX = w * 0.25 * CGFloat(attack)
                    let decayX = attackX + w * 0.25 * CGFloat(decay)
                    let sustainY = h * (1 - CGFloat(sustain))
                    let releaseX = w * 0.75 + w * 0.25 * CGFloat(release)

                    path.move(to: CGPoint(x: 0, y: h))
                    path.addLine(to: CGPoint(x: attackX, y: 0))
                    path.addLine(to: CGPoint(x: decayX, y: sustainY))
                    path.addLine(to: CGPoint(x: w * 0.75, y: sustainY))
                    path.addLine(to: CGPoint(x: releaseX, y: h))
                }
                .stroke(Color.green, lineWidth: 2)
            }
            .frame(height: 60)
            .background(Color.black.opacity(0.2))
            .cornerRadius(8)

            // Knobs for each parameter
            HStack(spacing: 16) {
                Knob(value: $attack, range: 0.001...5, label: "A", accentColor: .green)
                Knob(value: $decay, range: 0.001...5, label: "D", accentColor: .green)
                Knob(value: $sustain, range: 0...1, label: "S", accentColor: .green)
                Knob(value: $release, range: 0.001...10, label: "R", accentColor: .green)
            }
        }
        .padding()
        .background(Color.black.opacity(0.1))
        .cornerRadius(12)
    }
}
```

## Section Panel

```swift
struct SectionPanel<Content: View>: View {
    var title: String
    var accentColor: Color
    @ViewBuilder var content: () -> Content

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            // Section header
            HStack {
                Rectangle()
                    .fill(accentColor)
                    .frame(width: 4, height: 16)
                    .cornerRadius(2)

                Text(title.uppercased())
                    .font(.caption)
                    .fontWeight(.bold)
                    .foregroundColor(.gray)

                Spacer()
            }

            // Content
            content()
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 16)
                .fill(Color.black.opacity(0.2))
        )
    }
}
```

## Complete Synth Panel Layout

```swift
struct SynthPanelView: View {
    @ObservedObject var conductor: SynthConductor

    var body: some View {
        GeometryReader { geometry in
            let isCompact = geometry.size.width < 600

            if isCompact {
                // iPhone / compact layout
                ScrollView {
                    VStack(spacing: 16) {
                        oscillatorSection
                        filterSection
                        envelopeSection
                        effectsSection
                    }
                    .padding()
                }
            } else {
                // iPad / regular layout
                HStack(spacing: 16) {
                    VStack(spacing: 16) {
                        oscillatorSection
                        filterSection
                    }
                    .frame(maxWidth: .infinity)

                    VStack(spacing: 16) {
                        envelopeSection
                        effectsSection
                    }
                    .frame(maxWidth: .infinity)
                }
                .padding()
            }
        }
        .background(
            LinearGradient(
                colors: [Color(white: 0.15), Color(white: 0.1)],
                startPoint: .top,
                endPoint: .bottom
            )
        )
    }

    var oscillatorSection: some View {
        SectionPanel(title: "Oscillator", accentColor: .orange) {
            HStack(spacing: 24) {
                Knob(value: $conductor.frequency, range: 20...2000,
                     label: "FREQ", unit: "Hz", accentColor: .orange)
                Knob(value: $conductor.pulseWidth, range: 0...1,
                     label: "WIDTH", accentColor: .orange)
                WaveformPicker(selection: $conductor.waveform)
            }
        }
    }

    // ... other sections
}
```

## Color Scheme

```swift
extension Color {
    static let synthBackground = Color(white: 0.12)
    static let panelBackground = Color(white: 0.08)
    static let knobBody = Color(white: 0.25)

    // Section accents
    static let oscillatorAccent = Color.orange
    static let filterAccent = Color.cyan
    static let envelopeAccent = Color.green
    static let effectsAccent = Color.purple
    static let masterAccent = Color.red
}
```

## Animation Patterns

```swift
// Spring animation for controls
.animation(.spring(response: 0.3, dampingFraction: 0.7), value: value)

// Haptic feedback (iOS)
.sensoryFeedback(.selection, trigger: value)

// Scale on press
.scaleEffect(isPressed ? 0.95 : 1.0)
```

## Accessibility

```swift
.accessibilityLabel("Filter cutoff frequency")
.accessibilityValue("\(Int(cutoff)) Hertz")
.accessibilityAdjustableAction { direction in
    switch direction {
    case .increment:
        cutoff = min(cutoff + 100, 20000)
    case .decrement:
        cutoff = max(cutoff - 100, 20)
    @unknown default:
        break
    }
}
```
