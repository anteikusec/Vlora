# Vlora

High-Performance Android Download Manager, Stream Sniffer, and BitTorrent Client.

Vlora is an open-source, production-grade download accelerator engineered to deliver maximum network throughput, low memory footprint, and reliable background operations. Designed from the ground up as a modern alternative to legacy tools like ADM and 1DM, Vlora combines a Flutter user interface with an asynchronous Rust core engine compiled via the Android NDK.

---

## Architectural Overview

Vlora uses a layered architecture separating reactive presentation from high-throughput native disk and network execution:

```
+-------------------------------------------------------------------+
|               Flutter Material 3 UI (Dart / Riverpod)             |
|  - Active Queue & Progress Visualizer                             |
|  - Integrated Web Browser & Media Interceptor                     |
|  - Remote Control Web Dashboard (Local Network REST Server)       |
|  - Full Internationalization (English / Persian RTL)              |
+---------------------------------+---------------------------------+
                                  |
                   Dart FFI / Android JNI Bridge
                                  |
+---------------------------------v---------------------------------+
|                   Rust Core Engine (NDK Native)                   |
|  - Tokio Multi-threaded Async Runtime                             |
|  - Adaptive Multi-Segment HTTP/HTTPS Fetcher (Work-Stealing)      |
|  - Librqbit BitTorrent & Magnet Engine (DHT, uTP, PeX, LSD)       |
|  - Mmap Direct Memory-Mapped Disk I/O & POSIX fallocate           |
|  - Hardware-Accelerated Checksum Hasher (SHA-256 / SHA-1 / MD5)   |
|  - SQLite Persistence in WAL (Write-Ahead Logging) Mode           |
+-------------------------------------------------------------------+
```

---

## Core Capabilities

### 1. Multi-Segment HTTP/HTTPS Acceleration
- Dynamic chunk partitioning from 1 up to 32 concurrent streams per task.
- Work-stealing scheduling: idle or fast connections dynamically split remaining bytes of lagging connections.
- HTTP/2 multiplexing and persistent HTTP/1.1 connection pooling with custom keep-alive and TCP_NODELAY configurations.
- Automatic resume support with precise byte-range header validation.

### 2. Native BitTorrent and Magnet Engine
- Embedded pure-Rust BitTorrent client powered by Tokio and librqbit.
- Full support for magnet URIs and standalone `.torrent` binary files.
- Dual-stack peer wire communication over uTP (Micro Transport Protocol) and raw TCP.
- Distributed Hash Table (Kademlia DHT), Peer Exchange (PeX), and Local Peer Discovery (LSD).
- Automated tracker enrichment injecting verified public tier-1 announce servers into bare magnets.
- Selective file downloading within multi-file torrent archives.

### 3. Positional Zero-Copy Disk I/O
- Kernel-level pre-allocation via `fallocate` / `posix_fallocate` on supported filesystems (ext4/f2fs), preventing storage fragmentation and out-of-space mid-stream stalls.
- Zero-copy writes using memory-mapped buffers (`memmap2`) and asynchronous sequential flushing.
- Integrity verification pipeline supporting SHA-256, SHA-1, and MD5 algorithms with piece-level hash caching.

### 4. Smart Video Sniffer and HLS Extraction
- Embedded privacy-focused web browser with automated link sniffing.
- HTTP/M3U8 Master Playlist parser with variant resolution and bitrate detection (1080p, 720p, 480p, 360p).
- Automated DRM detection (Widevine, FairPlay, SAMPLE-AES) to alert users before queueing.
- Bulk link grabber and regex-powered TV episode / batch season extractor.

### 5. Remote Web Management Server
- Built-in lightweight HTTP REST daemon operating on a local port.
- PIN-protected HTML5 web dashboard allowing full remote management from desktop browsers over Wi-Fi.
- Real-time task inspection, link dispatch, pause, resume, cancel, and global bandwidth regulation.

### 6. Android Platform Integration
- Persistent foreground service keeping background downloads active without OS battery-killer termination.
- Dynamic notification controls featuring live download rates, progress bars, pause, and resume action triggers.
- Support for Scoped Storage and Android 11+ All Files Access (`MANAGE_EXTERNAL_STORAGE`).
- Deep-link handlers for `magnet:`, `.torrent` file association, and Android share sheet reception.

---

## Technical Specifications

| Component | Technology | Role |
| :--- | :--- | :--- |
| UI Framework | Flutter 3.x (Dart 3) | Cross-platform Material 3 responsive UI |
| State Management | Flutter Riverpod 2.6 | Unidirectional reactive state propagation |
| Core Engine | Rust (Edition 2021) | Concurrency, protocol parsing, and disk I/O |
| Async Runtime | Tokio 1.x (multi-thread) | Thread pool and event loop |
| Torrent Stack | librqbit / bittorrent | DHT, uTP, peer wire protocol |
| HTTP Client | Reqwest / Rustls | HTTP/1.1 and HTTP/2 pipeline |
| Disk Engine | memmap2 / libc POSIX | Zero-copy memory mapped writes |
| Persistence | SQLite via rusqlite (WAL) | State persistence across restarts |
| Local Server | Dart Shelf / IO HTTP Server | LAN Web dashboard and REST API |
| Android Native | Kotlin / Android NDK | Foreground services and platform bindings |

---

## Directory Structure

```
.
|-- android/                 # Android native project and Gradle setup
|   |-- app/                 # Application build config and native manifest
|   |   `-- src/main/jniLibs # Compiled Rust core shared libraries (.so)
|-- assets/                  # App branding, icons, and localized typography
|   |-- fonts/               # Noto Sans and Vazirmatn font families
|   `-- icon/                # Vector and raster application icons
|-- lib/                     # Flutter Dart codebase
|   |-- core/                # FFI bridges, services, models, themes, l10n
|   |   |-- ffi/             # C-ABI and Dart FFI bindings (VorloBridge)
|   |   |-- models/          # Task, progress, torrent, and media data models
|   |   `-- services/        # Notifications, permissions, storage, remote server
|   |-- features/            # Feature-first modular presentation and providers
|   |   |-- browser/         # Web browser, link grabber, video detector
|   |   |-- queue/           # Active downloads, segment visualizer, task cards
|   |   |-- settings/        # Speed limits, storage folder, network options
|   |   `-- torrent/         # Torrent manager, peer graphs, file selection
|   `-- main.dart            # Flutter application entry point
|-- release_apks/            # Pre-compiled, signed production release APKs
|-- rust/                    # Native Rust core crate (vorlo_core)
|   |-- src/                 # Engine, HTTP, Torrent, I/O, Database, FFI/JNI APIs
|   |-- tests/               # Integration and unit tests
|   `-- Cargo.toml           # Rust package configuration and dependencies
`-- test/                    # Automated Flutter unit and widget test suite
```

---

## Building from Source

### Prerequisites

Ensure the following tools are installed in your environment:
1. Flutter SDK (version 3.22.0 or higher)
2. Android SDK & NDK (NDK r26 or higher recommended)
3. Java Development Kit (JDK 17)
4. Rust toolchain (`rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android`)
5. `cargo-ndk` (`cargo install cargo-ndk`)

### 1. Build the Native Rust Core (Optional if pre-built .so are used)

The repository already contains pre-built native libraries in `android/app/src/main/jniLibs/`. If you wish to compile the Rust core from scratch:

```bash
cd rust

# Build for 64-bit ARM (arm64-v8a)
cargo ndk -t arm64-v8a -o ../android/app/src/main/jniLibs build --release

# Build for 32-bit ARM (armeabi-v7a)
cargo ndk -t armeabi-v7a -o ../android/app/src/main/jniLibs build --release

# Build for x86_64 (Emulators)
cargo ndk -t x86_64 -o ../android/app/src/main/jniLibs build --release
```

### 2. Run the Test Suites

Execute native Rust tests:
```bash
cd rust
cargo test
```

Execute Flutter Dart tests:
```bash
cd ..
flutter test
```

### 3. Build Production APKs

Generate release binaries split per ABI (recommended for smaller download footprints):

```bash
flutter build apk --release --split-per-abi
```

Or build a universal binary containing all native architectures:

```bash
flutter build apk --release
```

The resulting binaries are located in `build/app/outputs/flutter-apk/`.

---

## Pre-built Binaries

Pre-compiled and signed APK packages are available in the [`release_apks/`](release_apks/) directory:

- `Vlora-arm64-v8a.apk`: Optimized for modern 64-bit Android smartphones.
- `Vlora-armeabi-v7a.apk`: Backward-compatible package for older 32-bit Android devices.
- `Vlora-universal.apk`: Single bundle containing all native ABIs.
- `Vlora-x86_64.apk`: Targeted at Android x86_64 emulators and Intel/AMD tablets.

---

## Required Permissions

| Permission | Purpose |
| :--- | :--- |
| `INTERNET` | Downloading HTTP and BitTorrent network packets |
| `ACCESS_NETWORK_STATE` | Detecting Wi-Fi vs mobile cellular connections |
| `FOREGROUND_SERVICE` | Maintaining ongoing downloads when UI is in background |
| `FOREGROUND_SERVICE_DATA_SYNC` | Android 14 compliant foreground service categorization |
| `POST_NOTIFICATIONS` | Notification status bar speed and ETA display (Android 13+) |
| `MANAGE_EXTERNAL_STORAGE` | Saving files to user-defined directories outside sandbox |
| `WAKE_LOCK` | Preventing CPU sleep during active multi-gigabyte transfers |

---

## Contributing

Contributions are welcome. Please adhere to the following workflow:
1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/your-feature-name`.
3. Ensure all tests pass: `cargo test` and `flutter test`.
4. Run `flutter analyze` to ensure strict lint compliance.
5. Commit your changes and open a Pull Request.

---

## License

This project is licensed under the terms of the [MIT License](LICENSE).
