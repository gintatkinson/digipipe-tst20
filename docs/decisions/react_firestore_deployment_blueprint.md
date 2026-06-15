# React Firestore Hybrid Deployment Architecture Blueprint

This blueprint outlines the simplified architecture for the 3D topology visualization module. It supports a unified **Hybrid Flutter Shell + Embedded React** model for web and native desktop (macOS & Windows) deployments.

---

## 1. Architectural Strategy

To support both **Firestore SDK** and a future **Protobuf-backed API** (e.g., gRPC-web or REST with Protobuf serialization over WebSockets), the application utilizes a strict **Hexagonal Architecture (Ports and Adapters)**. The React 3D view is completely decoupled from the data binding layer.

```mermaid
graph TD
    subgraph UI ["User Interface Layer"]
        ReactUI[React 3D Canvas] --> Ports[Service Ports / Interfaces]
    end

    subgraph Adapters ["Adapters Layer (Swappable Bindings)"]
        Ports --> FirestoreAdapter[Firestore Web SDK Adapter]
        Ports --> ProtoAdapter[Protobuf gRPC-web Adapter]
    end

    subgraph Data ["Data & Server Layer"]
        FirestoreAdapter --> CloudFirestore{{Google Cloud Firestore}}
        ProtoAdapter --> ProtoBackend{{Rust Backend / Protobuf API}}
    end
    
    style UI fill:#2A2D34,stroke:#4C566A,stroke-width:2px,color:#ECEFF4
    style Adapters fill:#3B4252,stroke:#4C566A,stroke-width:2px,color:#ECEFF4
    style Data fill:#4C566A,stroke:#4C566A,stroke-width:2px,color:#ECEFF4
```

- **Loose Coupling**: React rendering layers bind exclusively to Ports (abstract TypeScript interfaces like `IDatabaseService`).
- **Swappable Bindings**: Swapping the database from Firestore to a Protobuf API involves only replacing the adapter binding (e.g., injecting `gRPCWebTopologyAdapter` instead of `FirestoreTopologyAdapter`), leaving all UI rendering code entirely unchanged.

---

## 2. Deployment Archetype 1: Standalone React Dev Sandbox (Vite/Express)

For local development, testing, and isolated UI work on the 3D topology view, the React module runs as a standalone Vite web project.

### 2.1 Developer Local Offline-First Persistence
The React module uses the Firebase Web SDK's offline capabilities (`IndexedDB`) to support disconnected testing:

```typescript
// src/services/firestore-init.ts
import { initializeApp } from 'firebase/app';
import { 
  initializeFirestore, 
  persistentLocalCache, 
  persistentMultipleTabManager,
  connectFirestoreEmulator
} from 'firebase/firestore';
import firebaseConfig from '../../firebase-applet-config.json';

const app = initializeApp(firebaseConfig);

// Initialize Firestore with persistent IndexedDB cache
export const db = initializeFirestore(app, {
  localCache: persistentLocalCache({
    tabManager: persistentMultipleTabManager()
  })
});

// Developer Sandboxing with local emulator
if (import.meta.env.DEV && import.meta.env.VITE_USE_EMULATOR === 'true') {
  connectFirestoreEmulator(db, 'localhost', 8080);
  console.log('Connected to local Firestore emulator (localhost:8080)');
}
```

---

## 3. Deployment Archetype 2: Embedded Webview (Production Desktop/Web)

In production, the Flutter application acts as the compile-target host for macOS, Windows, and Web.

### 3.1 Webview Integration
* **Desktop targets (macOS/Windows)**: Flutter compiles to a native C++ application and embeds the React 3D topology module using native desktop webview widgets (e.g., WebView2 on Windows, WebKit/WKWebView on macOS).
* **Web targets**: Flutter embeds the React build as a static micro-frontend using an `iframe` element wrapper loaded via Flutter's `HtmlElementView`.

### 3.2 Decoupled Synchronization via Firestore
Rather than serialization/IPC bridges, data synchronizes reactively:
* **Write Path**: Any user edit (dragging a node in React or typing in a Flutter form) updates the Firestore `/nodes/{nodeId}` document.
- **Reactive Repaint**: Both the Flutter shell (via Dart `snapshots()`) and the React canvas (via JS `onSnapshot()`) listen to database changes. When a coordinate changes, the 3D view repaints automatically in real-time.

---

## 4. Platform Mapping Strategy (React to Flutter)

To maintain design parity across the hybrid layers, the React components and hooks correspond directly to Flutter widgets and providers:

| React Concept | Flutter Equivalent | Description |
|---|---|---|
| **Vite Dev Server** | `flutter run` | Local development server |
| **TailwindCSS Classes** | `ThemeData` + Widget styling | Declarative visual styling |
| **React Context** | Riverpod `Provider` | Shared state and dependency injection |
| **Custom Hooks (`useAuth`)** | `Notifier` / `StateNotifier` | Encapsulated business and auth logic |
| **Firebase Web SDK** | `cloud_firestore` / `firebase_auth` | FlutterFire native plugins |
| **IndexedDB Cache** | SQLite Cache / Hive | Local persistence engines |
| **JS/Webview Interface** | Dart `JavascriptChannel` | Host-to-Client fallback communication channel |

---

## 5. Implementation Action Plan

```mermaid
flowchart TD
    subgraph P1 ["Phase 1: Local Sandbox Setup"]
        p1_1["Configure Vite Dev Server (1 hour)"] --> p1_2["Implement firestore-init with IndexedDB (30 mins)"]
    end
    subgraph P2 ["Phase 2: Hybrid Webview Integration"]
        p1_2 --> p2_1["Embed React view in Flutter WebView (1 hour)"] --> p2_2["Setup Firestore Reactive listeners (1 hour)"]
    end
    subgraph P3 ["Phase 3: Verification"]
        p2_2 --> p3_1["Verify Offline Caching & Sync (1 hour)"] --> p3_2["Run E2E Webview Tests (1 hour)"]
    end
    
    style P1 fill:#2A2D34,stroke:#4C566A,stroke-width:2px,color:#ECEFF4
    style P2 fill:#3B4252,stroke:#4C566A,stroke-width:2px,color:#ECEFF4
    style P3 fill:#434C5E,stroke:#4C566A,stroke-width:2px,color:#ECEFF4
```

1. **Step 1**: Set up the standalone React Vite configuration for the 3D topology canvas.
2. **Step 2**: Configure the `firestore-init.ts` wrapper with `persistentLocalCache` for IndexedDB cache management.
3. **Step 3**: Embed the React webview in the Flutter shell and configure Firestore snapshot synchronizations.
4. **Step 4**: Verify write synchronizations, offline cache fallbacks, and emulator redirection.
