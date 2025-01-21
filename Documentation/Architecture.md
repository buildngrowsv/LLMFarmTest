# LLMFarm Architecture Overview

## Core Components

### 1. Model Infrastructure
- **llmfarm_core**: Core Swift package handling model operations
- **Metal Acceleration**: GPU acceleration support for compatible devices
- **Model Support**: Multiple inference types including LLaMA, Gemma, GPT2, etc.

### 2. Key Features
- Local model execution on device
- RAG (Retrieval Augmented Generation) capabilities
- Various sampling methods (Temperature, TFS, Mirostat, etc.)
- Context state management
- Apple Shortcuts integration

### 3. Main Components
- **AI Class**: Main interface for model operations
  - Model initialization
  - Prediction handling
  - Context management
  
- **ModelInference**: Handles different model types
  - LLaMA
  - GPT2
  - RWKV
  - And others

- **ModelAndContextParams**: Configuration for model execution
  - Context size
  - Thread count
  - Metal usage
  - Special tokens handling

## Integration Points

### Adding New Capabilities
To add new capabilities like TTS:
1. Create new Swift package for the capability
2. Integrate with existing AI class
3. Add appropriate UI controls
4. Implement callback system for real-time processing

### Current Callback System
The system uses a callback mechanism for processing model outputs:
```swift
func mainCallback(_ str: String, _ time: Double) -> Bool
```
This can be extended for new features like TTS integration.

## Performance Considerations
- Metal acceleration for compatible devices
- Thread management for optimal performance
- Context size management for memory efficiency 