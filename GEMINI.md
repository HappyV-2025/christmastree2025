# Project: Christmas Express: Definitive Edition

This is a standalone, procedural 3D web application featuring a Christmas train scene. It is built using **Three.js** and **Web Audio API** without any external assets (textures, models, or audio files are all generated via code).

## 🚀 Running the Project

Since this is a client-side application with no build step:

1.  **Open `index.html`** directly in a modern web browser (Chrome, Edge, Firefox, Safari).
2.  Alternatively, use a local static server (e.g., `python -m http.server`, `live-server`, or VS Code's "Live Server" extension) to serve the directory.

## 📂 Project Structure

*   **`index.html`**: The monolithic source file containing:
    *   **HTML**: UI structure (Start screen, HUD, Settings modal).
    *   **CSS**: Styling for the UI overlay.
    *   **JavaScript**: The entire 3D engine, logic, and audio synthesis.
*   **`readme.md`**: Detailed functional specifications and feature list.
*   **`*_analysis.md`**: Documentation on specific algorithms (obstacle avoidance, camera positioning).

## 🛠️ Tech Stack & Conventions

*   **Engine**: Three.js (v0.160.0) loaded via CDN (ES Modules).
*   **Language**: Modern JavaScript (ES6+).
*   **Asset Strategy**: **Zero external assets**.
    *   **Geometry**: Procedural generation (Train, Track, Trees, Snow).
    *   **Textures**: Dynamic `CanvasTexture` generation (Smoke, Snowflakes, UI elements).
    *   **Audio**: Real-time synthesis using `AudioContext` (Oscillators, Noise Buffers).

## 🧩 Key Code Components (in `index.html`)

The JavaScript code is organized into a modular structure within the `<script type="module">` tag:

*   **`CFG` / `CONFIG`**: Global configuration object for physics, visual effects, and camera settings.
*   **`AudioEngine`**: Manages the Web Audio API context, generating BGM and SFX (whistle, chugging, fireworks).
*   **`CameraDirector`**: Handles intelligent camera movements (Drone Follow, Free Mode, Event transitions).
*   **`Train` Class**: Manages the locomotive and carriages, including wheel animation and physics-based movement along the `trackCurve`.
*   **`DecorManager`**: Uses `InstancedMesh` for high-performance rendering of repetitive environment objects (Trees, Sleepers, Snowmen).
*   **Particle Systems**: Custom shader-based systems for `Snow`, `Fireworks`, and `MagicTrail`.
*   **`AnimationSystem`**: The main game loop that coordinates updates across all subsystems.

## 🎮 Interaction

*   **Start**: Click "鸣笛发车" to initialize audio context and start the scene.
*   **Camera**:
    *   **Smart Mode**: Automatic cinematic angles.
    *   **Free Mode**: Drag to rotate, scroll to zoom (OrbitControls).
*   **Interactions**:
    *   Click **Train Head**: Toggle BGM.
    *   Click **Carriage**: Trigger bell sound and throw a gift.
    *   **UI Buttons**: Firework launch, settings menu, pause/play.
