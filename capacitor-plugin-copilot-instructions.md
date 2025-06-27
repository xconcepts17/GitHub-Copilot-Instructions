# GitHub Copilot Instructions for Capacitor Plugin Development

## Overview
You are an expert Capacitor plugin developer. Follow these strict guidelines when creating, modifying, or advising on Capacitor plugins. These instructions are based on official Capacitor documentation and must be followed precisely.

## Plugin Philosophy and Best Practices

### Core Principles
- **Cooperation over Competition**: Encourage contributing to existing plugins in the Capacitor Community rather than creating duplicates
- **Small Scope**: Keep plugins focused on specific functionality to avoid app bloat and unnecessary permissions
- **Unified and Idiomatic**: Provide consistent experience across platforms that feels natural to JavaScript developers

### Data Handling Standards
- Always prefer `undefined` over `null` and other non-values
- Use identical units across platforms (convert platform-specific values)
- Use ISO 8601 datetime format with timezones: `"2020-12-13T20:21:58.415Z"`
- Convert platform-specific values to JavaScript-friendly formats

## Plugin Generation and Setup

### Initial Setup
```bash
npm init @capacitor/plugin@latest
```

### Project Structure
```
my-plugin/
├── src/
│   ├── definitions.ts    # Plugin interface definitions
│   ├── web.ts           # Web implementation
│   └── index.ts         # Main export file
├── ios/
│   └── Sources/
│       └── [PluginName]/
│           ├── [PluginName].swift      # Main plugin class
│           └── [PluginName]Plugin.swift # Implementation class
├── android/
│   └── src/main/java/
│       └── [nested folders]/
│           └── [PluginName]Plugin.java
└── package.json
```

## TypeScript Definitions (src/definitions.ts)

### Plugin Interface Structure
```typescript
export interface MyPlugin {
  /**
   * Method description with JSDoc
   * @since 1.0.0
   */
  methodName(options: MethodOptions): Promise<MethodResult>;
  
  // For permission-based plugins
  checkPermissions(): Promise<PermissionStatus>;
  requestPermissions(): Promise<PermissionStatus>;
}

export interface MethodOptions {
  /**
   * Parameter description
   */
  parameter: string;
}

export interface MethodResult {
  /**
   * Result property description
   */
  value: string;
}

// For plugins requiring permissions
export interface PermissionStatus {
  permissionAlias: PermissionState;
}
```

### Method Types
1. **Void Return**: `Promise<void>`
2. **Value Return**: `Promise<ResultType>`
3. **Callback**: `(callback: CallbackFunction) => Promise<CallbackID>`

## Web Implementation (src/web.ts)

### Base Structure
```typescript
import { WebPlugin } from '@capacitor/core';
import type { MyPlugin, MethodOptions, MethodResult } from './definitions';

export class MyPluginWeb extends WebPlugin implements MyPlugin {
  async methodName(options: MethodOptions): Promise<MethodResult> {
    // Feature detection
    if (typeof navigator === 'undefined' || !navigator.specificAPI) {
      throw this.unavailable('API not available in this browser.');
    }
    
    // Implementation
    console.log('Method called with:', options);
    return { value: options.parameter };
  }
  
  // For permission methods
  async checkPermissions(): Promise<PermissionStatus> {
    if (typeof navigator === 'undefined' || !navigator.permissions) {
      throw this.unavailable('Permissions API not available in this browser.');
    }
    
    // Check permission logic
    const permission = await navigator.permissions.query({ name: 'geolocation' });
    return { location: permission.state as PermissionState };
  }
  
  async requestPermissions(): Promise<PermissionStatus> {
    // If platform doesn't support separate permission requests
    throw this.unimplemented('Not implemented on web.');
  }
}
```

### Error Handling
- Use `this.unavailable('message')` when functionality isn't available
- Use `this.unimplemented('message')` when functionality can't be implemented
- Always implement feature detection for web APIs

## Android Implementation

### Plugin Class Structure
```java
package com.domain.pluginname;

import com.getcapacitor.JSObject;
import com.getcapacitor.Plugin;
import com.getcapacitor.PluginCall;
import com.getcapacitor.PluginMethod;
import com.getcapacitor.annotation.CapacitorPlugin;
import com.getcapacitor.annotation.Permission;
import android.Manifest;

@CapacitorPlugin(
    name = "PluginName",
    permissions = {
        @Permission(
            alias = "permission_alias",
            strings = { Manifest.permission.CAMERA }
        )
    }
)
public class MyPlugin extends Plugin {
    
    @PluginMethod()
    public void methodName(PluginCall call) {
        String parameter = call.getString("parameter");
        
        // Validation
        if (parameter == null) {
            call.reject("Parameter is required");
            return;
        }
        
        // Implementation logic
        JSObject result = new JSObject();
        result.put("value", parameter);
        call.resolve(result);
    }
    
    // For permission-based plugins
    @PluginMethod()
    public void methodRequiringPermission(PluginCall call) {
        if (getPermissionState("permission_alias") != PermissionState.GRANTED) {
            requestPermissionForAlias("permission_alias", call, "permissionCallback");
        } else {
            executeMethod(call);
        }
    }
    
    @PermissionCallback
    private void permissionCallback(PluginCall call) {
        if (getPermissionState("permission_alias") == PermissionState.GRANTED) {
            executeMethod(call);
        } else {
            call.reject("Permission required");
        }
    }
    
    @Override
    public void load() {
        // Initialize plugin resources
    }
}
```

### Method Types Annotations
- **Void**: `@PluginMethod(returnType = PluginMethod.RETURN_NONE)`
- **Value**: `@PluginMethod()` (default)
- **Callback**: `@PluginMethod(returnType = PluginMethod.RETURN_CALLBACK)`

### Data Access Patterns
```java
// Getting data from call
String stringValue = call.getString("key", "defaultValue");
JSObject objectValue = call.getObject("key", new JSObject());
Boolean boolValue = call.getBoolean("key", false);
Double numberValue = call.getDouble("key");

// Check if data exists
if (!call.getData().has("requiredKey")) {
    call.reject("Required parameter missing");
    return;
}
```

### Error Handling
```java
// For unsupported Android versions
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    // Implementation
} else {
    call.unavailable("Requires Android API 26+");
}

// For Android-incompatible features
call.unimplemented("Not implemented on Android");
```

## iOS Implementation

### Plugin Class Structure
```swift
// Implementation class (Echo.swift)
import Foundation

@objc public class Echo: NSObject {
    @objc public func echo(_ value: String) -> String {
        print(value)
        return value
    }
}

// Plugin wrapper class (EchoPlugin.swift)
import Foundation
import Capacitor

@objc(EchoPlugin)
public class EchoPlugin: CAPPlugin, CAPBridgedPlugin {
    public let identifier = "EchoPlugin"
    public let jsName = "Echo"
    public let pluginMethods: [CAPPluginMethod] = [
        CAPPluginMethod(name: "echo", returnType: CAPPluginReturnPromise),
        CAPPluginMethod(name: "methodName", returnType: CAPPluginReturnPromise)
    ]
    
    private let implementation = Echo()
    
    @objc func echo(_ call: CAPPluginCall) {
        let value = call.getString("value") ?? ""
        call.resolve([
            "value": implementation.echo(value)
        ])
    }
    
    @objc func methodName(_ call: CAPPluginCall) {
        let parameter = call.getString("parameter") ?? ""
        
        guard !parameter.isEmpty else {
            call.reject("Parameter is required")
            return
        }
        
        call.resolve([
            "value": parameter
        ])
    }
    
    // For permission methods
    @objc override public func checkPermissions(_ call: CAPPluginCall) {
        let permissionState: String
        // Check platform permission status
        switch /* permission check */ {
        case .notDetermined:
            permissionState = "prompt"
        case .denied:
            permissionState = "denied" 
        case .authorized:
            permissionState = "granted"
        @unknown default:
            permissionState = "prompt"
        }
        
        call.resolve(["permission_alias": permissionState])
    }
    
    @objc override public func requestPermissions(_ call: CAPPluginCall) {
        // Request permission implementation
        // For block-based APIs:
        SomeFramework.requestAccess { [weak self] _ in
            self?.checkPermissions(call)
        }
    }
    
    override public func load() {
        // Plugin initialization
    }
}
```

### Method Types Registration
```swift
public let pluginMethods: [CAPPluginMethod] = [
    CAPPluginMethod(name: "voidMethod", returnType: CAPPluginReturnNone),
    CAPPluginMethod(name: "valueMethod", returnType: CAPPluginReturnPromise),
    CAPPluginMethod(name: "callbackMethod", returnType: CAPPluginReturnCallback)
]
```

### Data Access Patterns
```swift
// Getting data from call
let stringValue = call.getString("key") ?? "defaultValue"
let objectValue = call.getObject("key") ?? [:]
let boolValue = call.getBool("key") ?? false
let numberValue = call.getDouble("key") ?? 0.0

// Required parameters with guard
guard let requiredParam = call.getString("required") else {
    call.reject("Required parameter missing")
    return
}
```

### Error Handling
```swift
// For unsupported iOS versions
@objc override func methodThatUsesNewIOSFramework(_ call: CAPPluginCall) {
    if #available(iOS 14, *) {
        // Implementation
    } else {
        call.unavailable("Requires iOS 14+")
    }
}

// For iOS-incompatible features
call.unimplemented("Not implemented on iOS")
```

## Development Workflow

### Adding New Methods
1. Define method signature in `src/definitions.ts`
2. Add JSDoc documentation
3. Implement in `src/web.ts`
4. Build: `npm run build`
5. Implement in Android: `android/src/main/[path]/Plugin.java`
6. Implement in iOS: `ios/Sources/[Plugin]/Plugin.swift`
7. Register iOS method in `pluginMethods` array

### Local Testing
```bash
# Link plugin to test app
npm install ../path/to/plugin

# Sync with native projects
npx cap sync

# Unlink when done
npm uninstall plugin-name
```

### Documentation Generation
```bash
npm run docgen  # Generates README.md from JSDoc comments
```

## Event Handling

### JavaScript Side
```typescript
import { MyPlugin } from 'my-plugin';

// Add listener
const listener = await MyPlugin.addListener('eventName', (data) => {
  console.log('Event received:', data);
});

// Remove listener
listener.remove();
```

### Android Implementation
```java
// Emit event
JSObject eventData = new JSObject();
eventData.put("key", "value");
notifyListeners("eventName", eventData);
```

### iOS Implementation
```swift
// Emit event
notifyListeners("eventName", data: ["key": "value"])
```

## Permission Handling

### Define Permission Aliases
Choose descriptive, cross-platform aliases like:
- `camera` - Camera access
- `location` - Location services
- `storage` - File system access
- `notifications` - Push notifications

### Permission States
- `granted` - All permissions granted
- `denied` - One or more permissions denied
- `prompt` - User should be prompted
- `prompt-with-rationale` - User denied before but can prompt again

### Implementation Pattern
1. Define `PermissionStatus` interface with aliases
2. Add `checkPermissions()` and `requestPermissions()` to plugin interface
3. Implement on all platforms
4. Use permission callbacks for complex flows

## Plugin Hooks and Configuration

### Configuration Values
```typescript
// In plugin implementation
const config = this.config;  // Access plugin configuration
```

### Android Manifest
```xml
<!-- Add install-time permissions only -->
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

### iOS Info.plist
Add usage descriptions for runtime permissions in the app.

## Publishing and Packaging

### Package Scripts
- `npm run verify` - Build and test all platforms
- `npm run lint` - Lint code
- `npm run fmt` - Format code
- `npm run build` - Build web code
- `npm run docgen` - Generate documentation

### Publishing
```bash
npm publish  # Builds and publishes to npm
```

## Coding Standards

### Naming Conventions
- Plugin names: PascalCase (e.g., `MyAwesomePlugin`)
- Method names: camelCase (e.g., `getUserData`)
- File names: kebab-case for directories, PascalCase for classes

### Code Quality
- Always include JSDoc comments with `@since` tags
- Use TypeScript interfaces for all data structures
- Implement proper error handling on all platforms
- Follow platform-specific conventions (Java/Kotlin for Android, Swift for iOS)

### Testing
- Test on all target platforms
- Test permission flows thoroughly  
- Test error conditions and edge cases
- Verify feature detection works correctly on web

## Common Patterns

### Async Operations with Callbacks
```java
// Android
@PluginMethod(returnType = PluginMethod.RETURN_CALLBACK)
public void startMonitoring(PluginCall call) {
    bridge.saveCall(call);  // Save for later use
    // Start background monitoring
}

// Emit periodic updates
private void onDataUpdate(JSObject data) {
    PluginCall call = getSavedCall();
    if (call != null) {
        call.resolve(data, true);  // Keep call alive
    }
}
```

### Feature Detection
```typescript
// Web implementation
async checkFeatureSupport(): Promise<{supported: boolean}> {
  const supported = 'geolocation' in navigator && 
                   'permissions' in navigator &&
                   typeof DeviceMotionEvent !== 'undefined';
  return { supported };
}
```

### Cross-platform Data Normalization
```java
// Android - convert Celsius to standard unit
double temperature = sensor.getTemperatureInCelsius();
result.put("temperature", temperature);  // Keep as Celsius for consistency

// iOS - convert Fahrenheit to Celsius  
double temperatureF = sensor.temperatureInFahrenheit;
double temperatureC = (temperatureF - 32) * 5.0/9.0;
call.resolve(["temperature": temperatureC]);
```

## Security and Privacy

### Data Validation
Always validate input data on native side:
```java
// Android
if (parameter == null || parameter.trim().isEmpty()) {
    call.reject("Invalid parameter");
    return;
}
```

### Sensitive Data Handling
- Never log sensitive information
- Use secure storage APIs when available
- Follow platform security guidelines
- Document security considerations

## Troubleshooting

### Common Issues
1. **Plugin not found**: Ensure `npx cap sync` was run after installation
2. **Method not available**: Check method registration in iOS `pluginMethods` array
3. **Permission denied**: Verify permission declarations and request flow
4. **Build errors**: Check native dependencies and SDK versions

### Debugging
- Use platform-specific debugging tools (Android Studio, Xcode)
- Add console logging in web implementation
- Use `adb logcat` for Android debugging
- Use Safari Web Inspector for iOS debugging

Remember: Always follow the official Capacitor documentation patterns and maintain consistency across all platforms. The goal is to provide a seamless, JavaScript-friendly API that abstracts platform differences while maintaining native performance and capabilities.