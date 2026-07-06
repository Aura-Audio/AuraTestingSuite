The evolution of the **Aura Audio** suite—from pure JavaScript prototypes to **FAUST** (Functional Audio Stream) implementations, and finally to high-performance **C/WebAssembly** cores—presents a unique testing challenge. Because FAUST is a functional, domain-specific language specifically designed for real-time signal processing [[4]], and libraries like `faustwasm` allow compiling FAUST DSP code directly into WebAudio nodes [[5]], your testing stack must be polyglot and architecture-agnostic.

Below is the comprehensive **Project Plan for Implementing the Aura Audio Automated Testing Stack**, designed to unify testing across all three DSP paradigms (JS, FAUST, C/Wasm).

---

# 📋 Project Plan: Aura Audio Unified Testing Stack

## 🎯 Objective
Establish a multi-layered, automated testing pipeline that verifies DSP mathematical accuracy, Web Audio API routing, UI/Canvas rendering, and hardware I/O across all Aura Audio applications, regardless of the underlying DSP core (JS, FAUST, or C/Wasm).

## 🏗️ Architecture Matrix: Testing Across DSP Cores

| Testing Layer | Pure JS Edition | FAUST Edition | C / WebAssembly Edition |
| :--- | :--- | :--- | :--- |
| **Layer 1: DSP Core** | **Jest / Vitest** (Direct function testing) | **faustwasm** (Compile `.dsp` to Wasm, run test vectors in Node) | **CTest / Node Wasm** (Compile `.c` via Emscripten, run test vectors) |
| **Layer 2: Audio Graph** | **OfflineAudioContext** (Mock JS Worklet) | **OfflineAudioContext** (Mock `FaustAudioWorkletNode`) | **OfflineAudioContext** (Mock C Wasm Worklet) |
| **Layer 3: UI & Canvas** | **Playwright** (DOM & `ImageData` assertions) | **Playwright** (DOM & `ImageData` assertions) | **Playwright** (DOM & `ImageData` assertions) |
| **Layer 4: Hardware I/O** | **Playwright** (`getUserMedia` mocking) | **Playwright** (`getUserMedia` mocking) | **Playwright** (`getUserMedia` mocking) |

---

## 📅 Phase 1: Infrastructure & Tooling Setup (Weeks 1-2)
**Goal:** Establish the monorepo structure, install test runners, and configure the DSP compiler toolchains.

*   **Task 1.1: Monorepo Configuration**
    *   Set up a monorepo (e.g., using Turborepo or Nx) to house `apps/aura-js`, `apps/aura-faust`, and `apps/aura-wasm`.
    *   Configure a shared `packages/test-utils` library for common audio mocking functions.
*   **Task 1.2: E2E Framework Installation**
    *   Install **Playwright** at the root level.
    *   Configure `playwright.config.ts` to spin up local static servers for all three app variants simultaneously on different ports.
*   **Task 1.3: DSP Compiler Toolchains**
    *   **JS:** Configure Vitest/Jest for Node.js environment execution.
    *   **FAUST:** Install the FAUST compiler and `faustwasm` toolchain [[5]]. Create an npm script `npm run build:faust` that compiles `.dsp` files into Wasm/JS wrappers.
    *   **C/Wasm:** Set up the Emscripten SDK (emsdk) in the development environment and CI runners.

---

## 📅 Phase 2: Layer 1 - DSP Core Unit Testing (Weeks 3-4)
**Goal:** Verify the mathematical integrity of the DSP algorithms in isolation, without the overhead of the browser or Web Audio API.

*   **Task 2.1: JS Core Testing**
    *   Extract the math functions (e.g., LFO calculations, scaling multipliers) into pure, side-effect-free JS utility files.
    *   Write Jest tests asserting exact floating-point outputs for given inputs.
*   **Task 2.2: FAUST Core Testing**
    *   Write a Node.js script that uses `faustwasm` to compile the `.dsp` file into a Wasm module in memory.
    *   Instantiate the FAUST Wasm module, feed it a known test vector (e.g., a DC offset of `1.0` or a 440Hz sine wave), and assert that the output buffer matches the expected mathematical transformation defined in the FAUST code.
*   **Task 2.3: C/Wasm Core Testing**
    *   Implement the Node.js Wasm runner (as detailed in the previous response) to feed mock `Float32Arrays` into `dsp.c`.
    *   Add assertions for edge cases: NaN prevention, soft-clipping (`tanhf`) thresholds, and zero-crossing accuracy.

---

## 📅 Phase 3: Layer 2 - Audio Graph Integration Testing (Weeks 5-6)
**Goal:** Verify that the Web Audio API routing, Worklet bridges, and node connections function deterministically.

*   **Task 3.1: The "Golden Master" Audio Generation**
    *   Create a script that uses `OfflineAudioContext` to render 5 seconds of audio through each app's complete node graph.
    *   Use a deterministic input (e.g., an `OscillatorNode` generating a sweep from 20Hz to 20kHz).
    *   Export the rendered `AudioBuffer` to a `.wav` file. These become your **Golden Masters** (baseline truth files).
*   **Task 3.2: Automated Graph Assertions**
    *   Write integration tests that render the audio in a headless browser (via Puppeteer or Playwright) and compare the resulting `Float32Array` against the Golden Master.
    *   *Tolerance Threshold:* Allow a micro-variance (e.g., `epsilon = 1e-5`) to account for floating-point differences between local machines and CI runners.
*   **Task 3.3: Worklet Bridge Verification**
    *   Specifically test the `postMessage` bridge. Send configuration updates (e.g., "Change Engine 1 Tracks to 500") during the `OfflineAudioContext` render and assert that the audio output changes exactly at the expected sample frame.

---

## 📅 Phase 4: Layer 3 & 4 - E2E, UI, and Hardware Mocking (Weeks 7-9)
**Goal:** Simulate a real user interacting with the app, verifying visual feedback and hardware permissions.

*   **Task 4.1: Hardware I/O Mocking (The "Fake Mic")**
    *   Implement a global Playwright `addInitScript` that intercepts `navigator.mediaDevices.getUserMedia`.
    *   Inject a fake `MediaStream` generated by an internal `AudioContext` playing a pre-recorded test tone or speech sample. This ensures tests are deterministic and do not require physical microphones.
*   **Task 4.2: Visualizer & Canvas Testing**
    *   Write Playwright tests that wait for the UI to initialize, click "Start Microphone", and then use `page.evaluate()` to extract `ImageData` from the FFT and Oscilloscope canvases.
    *   Assert that the pixel arrays contain the expected colors (e.g., the purple `#bb86fc` oscilloscope line or the green `#aeea00` FFT bands) proving the visualizers are actively drawing audio data.
*   **Task 4.3: Recording & Export Verification**
    *   Automate the click on the "Start Recording" button.
    *   Intercept the resulting `Blob` download via Playwright.
    *   Parse the downloaded WAV/WebM file in the test runner and verify it contains valid audio headers and non-silent data.

---

## 📅 Phase 5: CI/CD Pipeline & Automation (Weeks 10-11)
**Goal:** Automate the entire stack to run on every Pull Request.

*   **Task 5.1: GitHub Actions Workflow Configuration**
    *   Create a matrix strategy in `.github/workflows/test.yml` to run tests against all three DSP cores.
    *   **Crucial Step:** Use the `mymindstorm/setup-emsdk` action for the Wasm build, and a custom Docker container or setup script for the FAUST compiler toolchain.
*   **Task 5.2: Headless Audio Dependencies**
    *   Linux CI runners do not have audio drivers by default. Add a step to install `libasound2` and `pulseaudio` (or use `Xvfb` with a dummy audio sink) so the headless Chromium instances can successfully initialize the `AudioContext` without crashing.
*   **Task 5.3: Visual Regression Baselines**
    *   Configure Playwright to automatically update and commit Canvas screenshot baselines (`toHaveScreenshot()`) when a specific label (e.g., `update-baselines`) is applied to a PR.

---

## 🚀 Execution Strategy & Milestones

| Milestone | Deliverable | Success Metric |
| :--- | :--- | :--- |
| **M1: Tooling** | Monorepo & CI Runners configured | `npm run build` successfully compiles JS, FAUST `.dsp`, and C `.c` files in CI. |
| **M2: DSP Core** | Layer 1 Unit Tests | 100% coverage of math functions; FAUST and C test vectors pass in Node.js. |
| **M3: Audio Graph** | Layer 2 Integration Tests | `OfflineAudioContext` renders match Golden Masters within `1e-5` tolerance. |
| **M4: E2E & UI** | Layer 3/4 Playwright Suite | Headless browser successfully mocks mic, draws to Canvas, and exports a valid WAV. |
| **M5: CI/CD** | Automated Pipeline | PRs are automatically blocked if DSP math drifts, routing breaks, or UI fails to render. |

## 💡 Pro-Tips for the Aura Audio Team

1.  **FAUST UI Parameters:** When testing the FAUST edition, remember that FAUST automatically generates UI metadata. Your Playwright tests should dynamically query the FAUST JSON metadata to ensure the web UI sliders correctly map to the underlying Wasm DSP parameters.
2.  **Flake Prevention:** Web Audio timing can be flaky in CI. Never use `setTimeout()` to wait for audio to process. Always use `OfflineAudioContext` for deterministic audio testing, and use Playwright's `waitForFunction()` to wait for specific Canvas pixel states or DOM text updates (like the dB meter changing from `-∞ dB`).
3.  **Cross-Core Parity:** The ultimate goal of this stack is **Parity Testing**. You can write a master test that feeds the exact same test vector into the JS, FAUST, and C/Wasm versions of AuraScaler, and asserts that all three engines produce the *exact same audio output*. This guarantees that as you migrate apps from JS -> FAUST -> Wasm, the sonic character remains identical.
