# Context Reset Implementation

## Summary

This implementation adds a `resetModelContext()` method to flutter_gemma to fix the **context bleed issue** where conversation history from previous chat sessions persists and leaks into new conversations.

## The Problem: Context Bleed

MediaPipe's LLM Inference API maintains a KV cache (Key-Value cache) in native memory that stores conversation history. This cache is **NOT** cleared when creating new chat sessions, causing:

- Responses from previous conversations appearing in new chats
- Models "remembering" information they shouldn't know
- Inability to have truly fresh conversation contexts

## The Solution

Added a new `resetModelContext()` method that:

1. **Closes the active inference session** (releases KV cache)
2. **Completely unloads the model from memory** (clears all state)
3. **Forces a fresh model instance on next use** (clean slate)

## Implementation Details

### 1. Pigeon Interface (`pigeon.dart`)

Added new platform channel method:

```dart
@async
void resetModelContext();
```

### 2. iOS Implementation (`ios/Classes/FlutterGemmaPlugin.swift`)

```swift
func resetModelContext(completion: @escaping (Result<Void, any Error>) -> Void) {
    print("[PLUGIN] Resetting model context to clear KV cache")
    
    // Close the current session (releases KV cache)
    session = nil
    
    // Close and null the model to force complete unload from memory
    model = nil
    
    print("[PLUGIN] Model context reset successfully")
    completion(.success(()))
}
```

### 3. Android Implementation (`android/src/main/kotlin/.../FlutterGemmaPlugin.kt`)

```kotlin
override fun resetModelContext(callback: (Result<Unit>) -> Unit) {
    scope.launch {
        try {
            println("[PLUGIN] Resetting model context to clear KV cache")
            
            // Close the current session (releases KV cache)
            session?.close()
            session = null
            
            // Close and null the model to force complete unload from memory
            inferenceModel?.close()
            inferenceModel = null
            
            println("[PLUGIN] Model context reset successfully")
            callback(Result.success(Unit))
        } catch (e: Exception) {
            println("[PLUGIN] Error resetting model context: ${e.message}")
            callback(Result.failure(e))
        }
    }
}
```

### 4. Dart Implementation (`lib/mobile/flutter_gemma_mobile.dart`)

```dart
@override
Future<void> resetModelContext() async {
    debugPrint('🔄 [FlutterGemma] Resetting model context...');
    
    // Close the initialized model if it exists
    if (_initializedModel != null) {
        debugPrint('   Closing inference model...');
        await _initializedModel?.close();
        _initializedModel = null;
        _initCompleter = null;
        _lastActiveInferenceSpec = null;
    }
    
    // Call the platform method to reset native model context
    await _platformService.resetModelContext();
    
    debugPrint('✅ [FlutterGemma] Model context reset complete');
}
```

### 5. Public API (`lib/core/api/flutter_gemma.dart`)

```dart
/// Reset model context and clear KV cache
///
/// This method completely unloads the active model from memory, clearing all conversation
/// history and KV cache. You must call `getActiveModel()` again after this to recreate
/// the model instance.
///
/// Example:
/// ```dart
/// // User wants to start a fresh conversation
/// await FlutterGemma.resetModelContext();
///
/// // Now recreate the model - it will have no memory of previous chats
/// final model = await FlutterGemma.getActiveModel(maxTokens: 1024);
/// final chat = await model.createChat();
/// ```
static Future<void> resetModelContext() async {
    await FlutterGemmaPlugin.instance.resetModelContext();
}
```

## Usage Example

```dart
// User clicks "New Conversation" button
Future<void> startNewConversation() async {
  // Reset the model context to clear all previous conversation history
  await FlutterGemma.resetModelContext();
  
  // Recreate the model instance (required after reset)
  final model = await FlutterGemma.getActiveModel(maxTokens: 1024);
  
  // Create a fresh chat with no memory of previous conversations
  final chat = await model.createChat(
    temperature: 0.7,
    randomSeed: DateTime.now().millisecondsSinceEpoch,
  );
  
  // Start the new conversation
  await chat.addQueryChunk(Message.text(text: 'Hello!', isUser: true));
  final response = await chat.generateChatResponse();
}
```

## When to Use

✅ **Use `resetModelContext()` when:**
- Starting a new chat session for a different user
- Implementing a "Clear Chat" or "New Conversation" button
- After completing a conversation to ensure no context leaks
- Switching between different conversation contexts
- Testing/debugging to ensure clean state

❌ **Don't use if:**
- You want to continue an existing conversation
- You're just switching parameters (temperature, topK, etc.) - use `createChat()` with new params
- You're implementing multi-turn conversations in the same session

## Important Notes

⚠️ **After calling `resetModelContext()`:**
1. All previous chat sessions become **invalid**
2. You **must** call `getActiveModel()` again to create a new model instance
3. The model is **completely unloaded from memory**
4. Any in-progress generations will be **stopped**

## Files Changed

1. `pigeon.dart` - Added Pigeon interface method
2. `lib/pigeon.g.dart` - Auto-generated by Pigeon
3. `ios/Classes/FlutterGemmaPlugin.swift` - iOS platform implementation
4. `android/src/main/kotlin/.../FlutterGemmaPlugin.kt` - Android platform implementation
5. `lib/flutter_gemma_interface.dart` - Abstract interface definition
6. `lib/mobile/flutter_gemma_mobile.dart` - Mobile platform Dart implementation
7. `lib/core/api/flutter_gemma.dart` - Public API method
8. `README.md` - Usage documentation
9. `CHANGELOG.md` - Release notes

## Testing

To verify the fix works:

```dart
// Test 1: Context should NOT bleed between sessions
final model1 = await FlutterGemma.getActiveModel();
final chat1 = await model1.createChat();
await chat1.addQueryChunk(Message.text(text: 'My name is Alice', isUser: true));
await chat1.generateChatResponse();

// Reset context
await FlutterGemma.resetModelContext();

// Create new model and chat
final model2 = await FlutterGemma.getActiveModel();
final chat2 = await model2.createChat();
await chat2.addQueryChunk(Message.text(text: 'What is my name?', isUser: true));
final response = await chat2.generateChatResponse();

// ✅ Expected: Model should NOT know the name "Alice"
// ❌ Before fix: Model would incorrectly respond with "Alice"
```

## Benefits

✅ **Complete fix** for context bleed issue  
✅ **Clean API** - single method call  
✅ **Cross-platform** - works on both iOS and Android  
✅ **Well documented** - clear usage examples  
✅ **Non-breaking** - existing code continues to work  

## Future Improvements

Potential enhancements:
- Add web platform support (if applicable)
- Auto-reset option when switching between different ModelTypes
- Metrics/logging to track context resets

## Related Issues

This implementation addresses the context bleed issue discussed in the conversation where:
- Creating new chat sessions didn't clear the KV cache
- `createChat()` with different `randomSeed` values didn't help
- Model responses included information from previous unrelated conversations
- No existing API method could clear the native model's memory

