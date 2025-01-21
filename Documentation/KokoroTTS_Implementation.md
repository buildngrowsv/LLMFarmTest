# Implementing Kokoro-82M TTS in LLMFarm

## Overview
Kokoro-82M is a compact (350MB) but powerful TTS model that can be integrated into LLMFarm for high-quality speech synthesis. The model supports both American and British English voices.

## Implementation Architecture

### 1. Core Components

#### Swift Package Structure
```swift
// KokoroTTS.swift
public class KokoroTTS {
    private var model: OpaquePointer?
    private var voicepack: [Float]
    private var device: String // "cpu" or "metal"
    
    public struct Config {
        let modelPath: String
        let voicepackPath: String
        let useMetalAcceleration: Bool
        let sampleRate: Int = 24000
    }
}
```

### 2. Model Integration

#### Dependencies
```swift
// Package.swift additions
dependencies: [
    .package(url: "https://github.com/apple/swift-numerics", from: "1.0.0"),
    .package(url: "https://github.com/hexgrad/Kokoro-82M", from: "0.19.0")
]
```

#### Core Functions
```swift
extension KokoroTTS {
    // Initialize model and load weights
    public func initialize(config: Config) throws
    
    // Generate speech from text
    public func generateSpeech(
        text: String,
        voice: Voice = .default,
        language: Language = .americanEnglish
    ) async throws -> AudioData
    
    // Handle phoneme generation
    private func generatePhonemes(text: String, language: Language) -> [Int32]
    
    // Process audio output
    private func processAudio(rawOutput: [Float]) -> AudioData
}
```

### 3. Voice Management

#### Voice Configuration
```swift
public enum Voice: String {
    case default = "af"      // 50-50 mix of Bella & Sarah
    case bella = "af_bella"
    case sarah = "af_sarah"
    case adam = "am_adam"
    case michael = "am_michael"
    case emma = "bf_emma"
    case isabella = "bf_isabella"
    case george = "bm_george"
    case lewis = "bm_lewis"
    case nicole = "af_nicole"
    case sky = "af_sky"
}

public enum Language {
    case americanEnglish // 'a' prefix voices
    case britishEnglish  // 'b' prefix voices
}
```

### 4. Integration with LLMFarm Core

#### AI Class Extension
```swift
extension AI {
    public var tts: KokoroTTS?
    
    public func enableTTS(config: KokoroTTS.Config) throws {
        tts = KokoroTTS()
        try tts?.initialize(config: config)
    }
    
    // Modify callback to include TTS
    public func mainCallback(_ str: String, _ time: Double) -> Bool {
        if let tts = self.tts {
            Task {
                try await tts.generateSpeech(text: str)
            }
        }
        return false
    }
}
```

## Implementation Steps

### 1. Model Setup
1. Download and integrate model files:
```bash
git clone https://huggingface.co/hexgrad/Kokoro-82M
```

2. Install dependencies:
```swift
// Add to project setup
dependencies: [
    "phonemizer",
    "espeak-ng",
    "torch",
    "transformers",
    "scipy",
    "munch"
]
```

### 2. Core Implementation

#### Model Loading
```swift
public func initialize(config: Config) throws {
    // 1. Load ONNX model
    guard let modelPath = Bundle.main.path(
        forResource: "kokoro-v0_19",
        ofType: "onnx"
    ) else {
        throw TTSError.modelNotFound
    }
    
    // 2. Initialize Metal context if available
    if config.useMetalAcceleration {
        try initializeMetalContext()
    }
    
    // 3. Load voice pack
    try loadVoicePack(at: config.voicepackPath)
}
```

#### Speech Generation
```swift
public func generateSpeech(
    text: String,
    voice: Voice,
    language: Language
) async throws -> AudioData {
    // 1. Generate phonemes
    let phonemes = generatePhonemes(text: text, language: language)
    
    // 2. Split into chunks for long text
    let chunks = splitIntoChunks(phonemes, maxLength: 510)
    
    // 3. Process each chunk
    var audioChunks: [AudioData] = []
    for chunk in chunks {
        let audio = try await processChunk(
            chunk,
            voice: voice
        )
        audioChunks.append(audio)
    }
    
    // 4. Combine chunks with silence
    return combineAudioChunks(audioChunks, silenceDuration: 0.4)
}
```

### 3. Performance Optimizations

#### Metal Acceleration
```swift
private func initializeMetalContext() throws {
    guard let device = MTLCreateSystemDefaultDevice() else {
        throw TTSError.metalUnavailable
    }
    
    // Setup compute pipeline
    let library = try device.makeDefaultLibrary()
    let function = try library.makeFunction(name: "tts_forward")
    computePipelineState = try device.makeComputePipelineState(
        function: function
    )
}
```

#### Batch Processing
```swift
private func processChunk(
    _ chunk: [Int32],
    voice: Voice
) async throws -> AudioData {
    // 1. Prepare input tensors
    let inputTensor = prepareInputTensor(chunk)
    let voiceTensor = prepareVoiceTensor(voice)
    
    // 2. Run inference
    let output = try await runInference(
        input: inputTensor,
        voice: voiceTensor
    )
    
    // 3. Post-process audio
    return processAudio(output)
}
```

## Usage Example

```swift
// Initialize TTS
let ttsConfig = KokoroTTS.Config(
    modelPath: "path/to/kokoro-v0_19.onnx",
    voicepackPath: "path/to/voices",
    useMetalAcceleration: true
)

try ai.enableTTS(config: ttsConfig)

// Generate speech
let text = "Hello, this is a test of the Kokoro TTS system."
if let tts = ai.tts {
    let audio = try await tts.generateSpeech(
        text: text,
        voice: .bella,
        language: .americanEnglish
    )
    // Play audio or save to file
}
```

## Performance Considerations

1. **Memory Management**
   - Chunk long text to manage memory usage
   - Clean up resources after generation
   - Use appropriate batch sizes

2. **Optimization**
   - Use Metal acceleration when available
   - Cache frequently used phonemes
   - Implement voice pack streaming for large files

3. **Error Handling**
   - Handle model loading failures
   - Manage audio processing errors
   - Implement proper cleanup 