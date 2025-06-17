# Bidirectional Communication Between HarmonyOS ArkTS and React Native JS: Principles and Practices

This article focuses on explaining the principles and practices of bidirectional communication between HarmonyOS ArkTS and React Native JS, covering the complete path from the JS side to the ArkTS side and then to the C++ glue layer.

---

## Understanding the Overall Architecture

- **JS Side (React Native Business Code):** The JS/TS files that developers write daily.
- **C++ Glue Layer:** The bridging layer adapted for HarmonyOS, responsible for message and data transmission, working with TurboModule/JSI and other mechanisms.
- **ArkTS Side:** The implementation of HarmonyOS native capabilities, pages, components, etc., responsible for the final business implementation.

A simplified communication flow diagram is as follows:

```
JS (RN) <==> C++ (Glue Layer) <==> ArkTS (Native)
```

---

## I. Full Flow of JS (RN) to ArkTS Communication

### 1.1 JS Side Calls TurboModule

**(1) Declare TurboModule Interface**

```javascript
// src/specs/v2/NativeCalculator.js
import type {TurboModule} from 'react-native/Libraries/TurboModule/RCTExport';
import {TurboModuleRegistry} from 'react-native';

export interface Spec extends TurboModule {
  add(a: number, b: number): Promise<number>;
}
export default TurboModuleRegistry.get<Spec>('RTNCalculator') as Spec | null;
```

**(2) Actual Call**

```javascript
import { RTNCalculator } from 'rtn-calculator';

const result = await RTNCalculator.add(3, 7);
// result should be 10, actually goes to native
```

### 1.2 C++ Glue Code Implements TurboModule

**(1) Glue Header File Generation (Example)**

```cpp
// generated/RTNCalculator.h
#pragma once
#include "RNOH/TurboModule.h"
namespace rnoh {
class RTNCalculator : public ArkTSTurboModule {
public:
  RTNCalculator(const Context ctx, const std::string name);
};
}
```

**(2) Method Implementation and Registration**

```cpp
// generated/RTNCalculator.cpp
#include "RTNCalculator.h"
namespace rnoh {
RTNCalculator::RTNCalculator(const Context ctx, const std::string name): ArkTSTurboModule(ctx, name) {
  methodMap_ = {
    { "add", { 2, [](facebook::jsi::Runtime& rt, facebook::react::TurboModule& turboModule, const facebook::jsi::Value* args, size_t count) {
      // Directly forward to ArkTS
      return static_cast<ArkTSTurboModule&>(turboModule).callAsync(rt, "add", args, count);
    }}}
  };
}
}
```

> **Note:** This is equivalent to forwarding the parameters to the ArkTS side, where ArkTS implements the business logic.

### 1.3 ArkTS Side Implements Capability

```Arkts
// entry/src/main/ets/turbomodule/CalculatorModule.ets
import { TurboModule } from '@rnoh/react-native-openharmony/ts';
import { TM } from '@rnoh/react-native-openharmony/generated/ts';
export class CalculatorModule extends TurboModule implements TM.RTNCalculator.Spec {
  add(a: number, b: number): Promise<number> {
    return Promise.resolve(a + b);
  }
}
```

### 1.4 Return Result to JS Side

ArkTS implements a Promise return, and the C++ glue automatically converts it to a JS Promise, allowing JS to obtain the final result.

---

## II. Full Flow of ArkTS to JS (RN) Communication

### 2.1 ArkTS Side Sends Events to JS Side (DeviceEventEmitter Example)

```Arkts
// ArkTS Side
this.ctx.rnInstance.emitDeviceEvent("customEvent", { foo: 123 });
```

### 2.2 C++ Glue Code Forwards Events

```cpp
// C++ Implementation
// Corresponding to the internal emitDeviceEvent method
void RNInstance::emitDeviceEvent(const std::string& eventName, const folly::dynamic& payload) {
  // Actually calls the JS side's DeviceEventEmitter.emit through JSI
  // Pseudo code example:
  auto jsCallback = ...; // Get the JSEmit function
  jsCallback(eventName, payload);
}
```

### 2.3 JS Side Listens for Events

```javascript
import { DeviceEventEmitter } from 'react-native';

DeviceEventEmitter.addListener('customEvent', (data) => {
  // data = { foo: 123 }
  // Handle messages sent from the ArkTS side here
});
```

---

## III. JS (RN) Side Actively Listens for Native Events

As above, the JS side listens for events actively pushed from the native side through DeviceEventEmitter or TurboModule's callback interface.

---

## IV. Direct Message Communication Between ArkTS and JS (Message Bus Mechanism)

**ArkTS Sends Message to C++ Side:**

```Arkts
// ArkTS
this.ctx.rnInstance.postMessageToCpp("SAMPLE_MESSAGE", { foo: "bar" });
```

**C++ Glue Layer Observer Listens:**

```cpp
class MyComponentInstance : public CppComponentInstance<...>, public ArkTSMessageHub::Observer {
public:
  MyComponentInstance(Context context)
    : CppComponentInstance(std::move(context)), ArkTSMessageHub::Observer(m_deps->arkTSMessageHub) {}
  void onMessageReceived(ArkTSMessage const& message) override {
    if (message.name == "SAMPLE_MESSAGE") {
      // Handle message.payload
      // Can then be forwarded to JS via emitDeviceEvent
    }
  }
};
```

**C++ Sends Message to ArkTS Side:**

```cpp
m_deps->rnInstance.lock()->postMessageToArkTS("ANOTHER_MESSAGE", { key: "value" });
```

**ArkTS Side Listens for C++ Messages:**

```Arkts
const unsubscribe = rnInstance.cppEventEmitter.subscribe("ANOTHER_MESSAGE", (value: object) => {
  // value = { key: "value" }
  unsubscribe();
});
```

---

## V. Core Points Summary

- For communication between RN (JS) and ArkTS, the **C++ glue layer is the bridge**. Developers should mainly focus on the API usage on both ends.
- **JS side calls TurboModule method → C++ glue forwards → ArkTS implements capability → Promise returns to JS**
- **ArkTS/Native side sends events → C++ glue forwards → DeviceEventEmitter/Event Bus → JS side listens and responds**
- The **message bus mechanism** supports bidirectional custom event/message passing, suitable for complex scenarios.

---

## Conclusion

As long as you understand that "**all communication must go through the C++ glue layer**" and make good use of TurboModule/DeviceEventEmitter/Message Bus, bidirectional communication between HarmonyOS ArkTS and React Native JS will become very clear and simple!