# Extension Architecture

This document provides an overview of how **GitHub Copilot for Xcode** is organised and how the main pieces work together.

## Overview

The project is split into several targets. The **Copilot for Xcode** application bundles the editor extension and the XPC service. A separate **ExtensionService** process implements all of the functionality and communicates with the extension through a helper called **CommunicationBridge**.

At runtime, Xcode invokes the source editor extension whenever a user triggers Copilot commands. The extension forwards the editor state to the background XPC service. Most of the business logic lives in the Swift packages `Core` and `Tool`, which are shared by the host app and the service.

```
Xcode ─┬─ EditorExtension ──► CommunicationBridge ──► ExtensionService
       │
       └─ Host App (settings & UI)
```

The repository also contains a small Node project under `Server/` used for the web based diff view and terminal.

## Important components

### Copilot for Xcode (Host App)
- Entry point located in `Copilot for Xcode/App.swift`.
- Provides the settings UI and launches the editor extension or chat window depending on command line options.
- Starts or stops the XPC service when the application terminates.

### EditorExtension
- Located in the `EditorExtension/` directory.
- Implements Xcode source editor commands such as *Get Suggestions* or *Open Chat*.
- Each command gathers the current editor content and forwards it to the XPC service using `XPCExtensionService`.

### ExtensionService
- A background process defined in `ExtensionService/`.
- Hosts `XPCService` which contains the implementation of the features (chat, suggestions, realtime suggestions, etc.).
- Communicates back to the extension through `XPCCommunicationBridge`.

### CommunicationBridge
- Lives in `CommunicationBridge/`.
- Simple helper process that keeps the XPC connection open and relaunches the service if needed.

### Core and Tool packages
- `Core/` contains the Swift code for features such as `ChatService`, the GUI controllers and workspace management.
- `Tool/` exposes shared utilities and service providers including the `XPCShared` module which defines the XPC interfaces.

### Server
- TypeScript project located in `Server/`.
- Bundles a small web application used for diff views and integrated terminal support.

## Common logic

1. **XPC communication** – `XPCExtensionService` (from `Tool/Sources/XPCShared`) manages the connection to the background service. Commands in the editor extension call methods on this object.
2. **Service orchestration** – `Service` (inside `Core/Sources/Service`) coordinates realtime suggestions, workspace handling and the GUI. It is started by the `ExtensionService` app delegate.
3. **Chat and AI features** – `ChatService` (under `Core/Sources/ChatService`) interacts with the GitHub backend. It keeps track of conversation history and handles code edits suggested by the agent.
4. **Launch agent** – A launch agent ensures the background services can run when Xcode or the host app are active.

Together these pieces provide the Copilot experience in Xcode: suggestions are requested by the extension, processed in the service and presented back inside the editor or through the chat UI.
