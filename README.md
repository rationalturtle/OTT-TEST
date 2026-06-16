# OTT-Pipeline-TEST
OTT-Pipeline
OTT VOD Pipeline & Playback Dashboard

A serverless Video-on-Demand (VOD) transcoding pipeline that bridges Google Cloud Platform (GCP) and Amazon Web Services (AWS). It processes raw video uploads into an HLS adaptive bitrate (ABR) ladder and serves them to a custom diagnostic web player.

How It Works

The architecture relies on event-driven serverless compute to handle the cross-cloud packaging workflow:

Ingest: Raw media assets (video, alternate audio tracks, and subtitles) are uploaded to a GCP Cloud Storage bucket.

Orchestration: GCP Eventarc detects the upload and triggers a Cloud Run Python container. The function verifies that all necessary sidecar files are present before proceeding.

Cross-Cloud Transit: The Cloud Run function generates temporary, short-lived IAM Signed URLs for the GCP assets, allowing AWS to securely pull the files without public exposure.

Transcoding: AWS Elemental MediaConvert is triggered via Boto3. It pulls the assets and packages them into a strict Apple-compliant HLS ABR ladder (1080p, 720p, 480p, plus WebVTT). H.264 profiles are dynamically set to AUTO to handle variable input framerates without failing.

Storage: The packaged .m3u8 and .ts files are deposited into an AWS S3 origin bucket.

The Web Player (Command Center)

The frontend is a custom diagnostic dashboard built with vanilla JavaScript, Tailwind CSS, and Google's Shaka Player.

It specifically demonstrates client-side DRM and token handling fundamentals:

Network Interception: Overrides Shaka Player's default networking engine to inject an X-OTT-Access-Token header into every outgoing HTTP fetch request.

Telemetry: Pulls real-time stats from the Shaka engine to display current resolution, bandwidth estimates, and buffer health.

Tech Stack

Frontend: HTML/JS, Tailwind, Shaka Player API

Backend: Python 3, Boto3

GCP: Cloud Storage, Eventarc, Cloud Run, IAM Credentials API

AWS: Elemental MediaConvert, S3
