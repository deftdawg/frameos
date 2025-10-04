# FrameOS Architecture

This document outlines the architecture of the FrameOS system and proposes a new approach for supporting ESP32-based "NeoFrame" devices.

## 1. Existing Architecture

The current FrameOS system is composed of three main components: a frontend UI, a backend control server, and the edge node software (FrameOS) that runs on the display-powering hardware, typically a Raspberry Pi.

### Components

*   **Frontend (UI)**: A React-based web application that allows users to define and configure "scenes". A scene is a collection of logical blocks (apps, data sources, schedules, etc.) that determine what is rendered on the frame's display. The user interacts with this UI in their web browser.
*   **Backend (Control Server)**: A Python FastAPI application, typically run in a Docker container. It serves the frontend, provides an API for managing frames and scenes, generates Nim source code from the scene definitions, and orchestrates the deployment of this code to the edge nodes.
*   **FrameOS (Edge Node)**: The software that runs on the edge device (e.g., Raspberry Pi). It's built on NixOS (for Raspberry Pi) and consists of a core application written in the Nim programming language. It is responsible for compiling the deployed scene code, executing it to render an image, and driving a physical display.

### Diagrams

#### System Architecture

```mermaid
graph TD
    subgraph "User's Browser"
        A["Frontend UI"]
    end

    subgraph "Control Server (Docker)"
        B["Backend API (Python/FastAPI)"]
        C["Code Generator (scene_nim.py)"]
    end

    subgraph "Edge Node (Raspberry Pi)"
        D["FrameOS (Nim Application)"]
        E["SSH Server"]
        F["Web Server (for previews)"]
        G["Display Driver"]
        H["Physical Display"]
    end

    User -- "Interacts" --> A
    A -- "API Calls (REST/WebSocket)" --> B
    B -- "Generates Code" --> C
    B -- "Manages via SSH" --> E
    E -- "Controls" --> D
    D -- "Renders to" --> G
    G -- "Drives" --> H
    D -- "Serves Preview Image" --> F
    F -- "HTTP GET /image" --> A
```

#### Deployment Sequence Diagram

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend
    participant "Frame (Edge Node)"

    User->>Frontend: "Designs scene and clicks ""Deploy"""
    Frontend->>Backend: "POST /api/deploy (with scene JSON)"
    Backend->>Backend: "Enqueue deploy task"
    Note over Backend: "Scene JSON is used to generate Nim source code."
    Backend->>Backend: "Create build archive (.tar.gz)"
    Backend->>"Frame (Edge Node)": "Connect via SSH"
    Backend->>"Frame (Edge Node)": "Upload build archive"
    Backend->>"Frame (Edge Node)": "Execute `tar` to extract archive"
    Backend->>"Frame (Edge Node)": "Execute `make` to compile Nim code"
    Note over "Frame (Edge Node)": "Nim code is compiled into a native binary on the device."
    "Frame (Edge Node)"-->>Backend: "Compilation complete"
    Backend->>"Frame (Edge Node)": "Update symlinks to new release"
    Backend->>"Frame (Edge Node)": "Restart `frameos` service"
    "Frame (Edge Node)"->>"Frame (Edge Node)": "New `frameos` binary executes the scene logic"
```

## 2. Proposed Architecture for NeoFrame Support (Virtual Edge Node Approach)

This revised proposal adopts the user's suggestion to leverage the existing architecture by creating a "virtual" edge node that runs on the control server. This approach minimizes architectural changes and reuses the battle-tested deployment and rendering logic.

### Phase 1: Virtual Edge Node

#### Concept
Instead of rendering on the control server's backend process, we will run the existing `frameos` Nim application in a Docker container on the control server itself. This "virtual" edge node will be configured to render scenes and then, instead of driving a physical display, it will HTTP POST the rendered image to the target NeoFrame device.

#### Changes Required
1.  **Virtual Device Driver in FrameOS:**
    *   A new "device driver" will be created within the `frameos` Nim application.
    *   When this driver is selected (e.g., `device = "virtual-neoframe"` in `frame.json`), the `drivers.render(image)` function will not attempt to write to a physical display.
    *   Instead, it will take the rendered `image`, encode it to the format expected by the NeoFrame (e.g., PNG or BMP), and HTTP POST it to an upload URL (e.g., `http://<neoframe-ip>/upload`), which will be part of the device's configuration.

2.  **Control Server SSH Port Configuration:**
    *   The backend will be updated to allow specifying a custom SSH port when defining a frame. This allows the control server to connect to the virtual edge node container running locally on a non-standard port (e.g., 2222).

3.  **Deployment Workflow:**
    *   A user wanting to drive a NeoFrame will first start a `frameos` Docker container on their control server, mapping a unique SSH port (e.g., 2222) to the container's port 22.
    *   In the UI, they will add a new frame, providing the control server's IP, the custom SSH port, and credentials for the container.
    *   The scene will be designed as usual. The only difference is the device configuration will specify the `virtual-neoframe` driver and the target NeoFrame's IP address for the upload.
    *   The deployment process remains identical: the control server will SSH into the local container, upload the code, compile it, and restart the `frameos` service inside the container. The `frameos` service will then handle the rendering and forwarding to the actual NeoFrame.

#### Phase 1 Architecture Diagram
```mermaid
graph TD
    subgraph "User's Browser"
        A["Frontend UI"]
    end

    subgraph "Control Server Host"
        subgraph "Control Server (Docker)"
            B["Backend API (Python/FastAPI)"]
        end
        subgraph "Virtual Edge Node (Docker)"
            D["FrameOS (Nim Application)"]
            E["SSH Server (Port 2222)"]
        end
    end

    subgraph "NeoFrame (ESP32)"
        K["Web Server (for upload)"]
        M["Physical Display"]
    end

    User -- "Interacts" --> A
    A -- "API Calls" --> B
    B -- "Deploys to (SSH)" --> E
    E -- "Controls" --> D
    D -- "Renders image and..." --> D
    D -- "...posts to" --> K
    K -- "Updates" --> M
```

### Phase 2: Multi-Device Support (Future Enhancement)

To further optimize this, the `frameos` application could be enhanced to support multiple target devices from a single instance. This would eliminate the need to run one container per NeoFrame.

*   The `frame.json` would be extended to support a list of devices.
*   The `frameos` application would iterate through this list, rendering and uploading to each target device.

This would be a more invasive change but could significantly improve scalability for managing many virtual frames.