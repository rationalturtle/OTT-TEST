# # Secure OTT Edge Pipeline
A deterministic, event-driven Video-on-Demand (VOD) pipeline that bridges **Next.js (Payload CMS)**, **Google Cloud Platform (GCP)**, and **Amazon Web Services (AWS)**.
This architecture processes raw mezzanine video uploads into secure HTTP Live Streaming (HLS) Adaptive Bitrate (ABR) ladders, protecting them behind a custom, sub-millisecond cryptographic edge firewall.

## 📐 System Architecture
The ecosystem splits responsibility across distinct cloud environments to achieve local entropy minimization (Minimax Engineering).

### 1\. Ingestion Plane (Payload CMS & Next.js)
* **Relational Topology:** Content operators upload media packages directly through a custom Payload CMS v3 dashboard backed by PostgreSQL.
* **Direct-to-Cloud (D2C):** Pre-save lifecycle hooks sanitize filenames and stream multi-gigabyte files (primary .mp4 and sidecar .srt/.wav tracks) directly to a Google Cloud Storage input bucket using a flat namespace hash.

### ⠀2. Multi-File Orchestration (GCP Eventarc & Cloud Run)
* **The Pacifier Pattern:** GCP Eventarc detects uploads and triggers a Python Cloud Run instance. The script deterministically checks if all sibling files are present. If a multi-part upload is still streaming, it safely goes to sleep to prevent pipeline race conditions.
* **Atomic Locks:** Once all assets arrive, the script claims a metadata lock (if_generation_match=0), generates short-lived IAM Signed URLs, and POSTs the transcoding schema to AWS.

### ⠀3. Compute Transcoding Engine (AWS MediaConvert)
* **The Factory:** AWS Elemental MediaConvert pulls the GCP assets securely and transcodes the video into an Apple-compliant HLS ABR ladder (1080p, 720p, 480p).
* **Storage:** Packaged .m3u8 manifests and .ts segments are deposited into an S3 Origin bucket, secured by "Block All Public Access" and strict Origin Access Control (OAC).

### ⠀4. Zero-Trust Edge Security (CloudFront)
* **CMS Token Minting:** Payload CMS utilizes a cookie-authenticated endpoint to mint temporary 2-part JWTs (payload.signature) signed with HMAC-SHA256 and a strict 15-minute Time-To-Live (TTL).
* **1-Millisecond Firewall:** A custom CloudFront ES5 Viewer-Request Function intercepts incoming network traffic. In ~0.08 milliseconds, it mathematically validates the HMAC signature and TTL. Invalid or expired requests are dropped with a 403 Forbidden before they ever reach the cache.

### ⠀5. Telemetry & Post-Mortem (AWS EventBridge)
* **Automated Garbage Collection:** AWS EventBridge detects COMPLETE or ERROR transcoding states and routes a secure webhook back to Payload CMS.
* **State Sync:** The Next.js backend updates the CMS database status to "Ready", logs the CDN playback URL, and cleans up the initial lock files from GCP.

## ⠀📺 The Command Center (Interactive Web Player)
The frontend is a completely stateless, decoupled diagnostic dashboard hosted on GitHub Pages, built with vanilla JavaScript, Tailwind CSS, and Google's Shaka Player.
* **Network Interception:** Overrides Shaka Player's default networking engine to inject the X-OTT-Access-Token JWT into every outgoing HTTP fetch request for seamless edge authentication.
* **Real-Time Telemetry:** Polls internal Shaka engine buffers to display active Adaptive Bitrate (ABR) ladders, estimated bandwidth, dropped frames, and active codecs in a live terminal UI.
* **Live Topology Academy:** Features a D3.js physics-driven relational graph outlining the system's architectural modules.

## ⠀🛠️ Tech Stack
**Frontend & Control Plane**
* HTML/JS, Tailwind CSS
* Shaka Player API, D3.js (Physics Engine)
* Next.js (React), Payload CMS v3, PostgreSQL

⠀**Google Cloud Platform (GCP)**
* Cloud Storage (Ingest)
* Eventarc (Event Routing)
* Cloud Run (Python 3 Orchestrator)
* IAM Credentials API

⠀**Amazon Web Services (AWS)**
* Elemental MediaConvert (Transcoding)
* S3 (Origin Storage)
* CloudFront & CloudFront Functions (CDN & Edge Security)
* EventBridge (Telemetry Hooks)
