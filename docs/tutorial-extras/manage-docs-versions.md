---
sidebar_position: 1
---

# Tích hợp với react native

**Installation**:

- `npm install react-native-sdk-tracking @react-native-async-storage/async-storage @react-native-community/netinfo`

## Create your first React Native

Create a file at `src/App`:

```jsx title="src/App"
import React, { useEffect } from "react";
import { VisitorTracker } from "react-native-sdk-tracking";

export default function App() {
  useEffect(() => {
    const startTracking = async () => {
      await VisitorTracker.init({
        token: "your-token",
        apiUrl: "your-api-url",
      });
      const md5 = VisitorTracker.getMd5();
    };
    startTracking();
  }, []);

  return (
    <View>
      <Text>Result</Text>
    </View>
  );
}
```
