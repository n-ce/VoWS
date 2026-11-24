HEVC Watch Together

A peer-to-peer video player that streams locally owned HEVC files to another person, synchronizing playback between two devices without uploading the original media or transcoding it.

1. Project principles

- Original media preservation: Transmit original encoded HEVC samples without decoding and re-encoding at the sender.
- Local-first: The host selects and plays a file from their local device. No permanent media upload or centralized media storage.
- Synchronized playback: The host controls playback, and the recipient follows the same playback timeline.
- Adaptive buffering: Both sender and receiver actively manage buffering, throughput, backpressure and playback continuity.
- Minimal infrastructure: Use peer-to-peer communication wherever possible. Servers should provide coordination and connectivity assistance, not media processing.
- Chromium-first: Target the latest stable Google Chrome on both devices.

2. Runtime and platform

- Bun 1.44+ as the JavaScript runtime, package manager and development environment.
- TypeScript throughout the application.
- Latest stable Google Chrome as the initial supported browser.
- Browser-native media APIs wherever practical.
- WebRTC for peer-to-peer communication.
- WebSocket for session signaling and coordination.

3. Core functional requirements

3.1 Local media playback

- Allow the host to select a local video file.
- Focus primarily on HEVC / H.265 video playback.
- Support 1080p HEVC as the initial target.
- Read and demux media locally.
- Preserve original encoded video samples and codec configuration.
- Do not transcode, resize or re-encode the original media.
- Preserve audio without transcoding where the recipient supports its codec.
- Extract and preserve presentation timestamps, decoding timestamps and sample durations.
- Handle keyframes, codec initialization data and stream boundaries correctly.

The original file must remain untouched.

3.2 Watch sessions

- The host creates a session.
- The application generates a shareable joining link.
- A recipient joins using the same application.
- The host acts as the authoritative playback controller.
- The recipient receives the media stream and follows the host's playback state.
- A session must handle connection loss and reconnection gracefully.

No account or permanent media upload should be required for the initial implementation.

3.3 Synchronized playback

The host controls:

- Play
- Pause
- Seek
- Playback speed
- Restart

The recipient should follow these operations with minimal delay.

Requirements:

- Maintain a shared playback timeline.
- Synchronize audio and video.
- Correct playback drift without unnecessary visible interruptions.
- Handle seeks by resetting or repositioning the relevant demuxing, transfer and decoding pipelines.
- Prevent stale buffered samples from being presented after a seek.
- Distinguish network buffering from an intentional host pause.

4. Adaptive buffering

Adaptive buffering is a core system requirement, not an optional optimization.

Both sender and receiver must actively manage their respective pipelines.

4.1 Sender-side buffering

The sender reads media from a local file and prepares encoded samples for transmission.

It must:

- Read ahead sufficiently to prevent disk I/O from interrupting playback.
- Adapt file read-ahead according to transfer throughput and receiver feedback.
- Manage demuxing and outgoing sample queues.
- Apply backpressure when the network cannot accept more data.
- Avoid unbounded memory growth.
- Prioritize samples according to playback deadlines.
- Maintain sufficient data ahead of the current playback position.
- Recover gracefully from temporary network congestion.
- Avoid unnecessarily reading or transmitting large portions of the file that are not needed yet.

Sender buffering should account for local file I/O, demuxing, network transmission and pending acknowledgements or receiver feedback.

4.2 Receiver-side buffering

The receiver must maintain enough buffered media to sustain uninterrupted playback.

It must:

- Monitor incoming throughput and buffer occupancy.
- Maintain a configurable playback buffer.
- Adapt its target buffer duration to network conditions.
- Manage incoming samples, demuxed media, decoding queues and presentation queues.
- Avoid decoding too far ahead.
- Detect impending buffer exhaustion.
- Request additional data or adjust delivery pacing as necessary.
- Recover from network interruptions without corrupting playback state.
- Discard obsolete buffered data following a seek.
- Avoid excessive memory consumption.

4.3 Adaptive buffer coordination

Sender and receiver must cooperate rather than operate independently.

The receiver should communicate relevant information to the sender, including:

- Current buffered duration.
- Playback position.
- Buffer health.
- Incoming throughput.
- Decoder queue pressure.
- Playback stalls.
- Data demand.

The sender uses this feedback to adjust read-ahead, transmission pacing and queue management.

The objective is to maintain smooth playback with the smallest practical startup delay and without excessive buffering latency.

4.4 Buffering objectives

- Avoid unbounded queues on either device.
- Avoid unnecessary retransmission.
- Minimize startup buffering.
- Prevent receiver starvation.
- Prevent sender memory exhaustion.
- Recover from fluctuating bandwidth.
- Keep playback synchronization independent of buffering decisions.

5. Media transport

Use WebRTC DataChannels for transmitting original encoded media samples.

Requirements:

- Reliable delivery of required media data.
- Explicit sample framing and stream identification.
- Preserve timestamps, durations and sample boundaries.
- Handle large transfers through bounded chunks.
- Implement application-level flow control.
- Monitor DataChannel "bufferedAmount" and apply backpressure.
- Separate playback control messages from bulk media transfer.
- Support connection establishment through signaling.
- Support STUN and TURN for connectivity when direct peer-to-peer connections fail.

The application must not assume that WebRTC automatically solves media buffering or playback synchronization.

6. Display and rendering

The initial target is 1920 × 1080 media.

Different recipient screen dimensions and aspect ratios must not require modifying the source video.

Requirements:

- Preserve the original encoded resolution.
- Scale video locally according to available display space.
- Support aspect-ratio-aware rendering.
- Support contain and cover presentation modes.
- Handle letterboxing and cropping locally.
- Avoid transmitting screen-specific crops or resized media.
- Keep the original media dimensions independent of CSS display dimensions.

Display adaptation is a rendering responsibility, not a media-transcoding responsibility.

7. Codec and browser compatibility

Latest stable Chrome is the initial target on both devices.

However, browser version alone does not guarantee hardware or operating-system HEVC support.

Requirements:

- Inspect the source codec configuration.
- Detect HEVC decoder availability before joining playback.
- Verify support for the specific profile, level, bit depth and chroma format.
- Detect supported audio codecs.
- Validate decoder configuration using WebCodecs capability checks.
- Fail early with a clear compatibility message when decoding is unavailable.
- Never silently transcode unsupported media.

8. Technical architecture

Initial technologies to investigate:

Responsibility| Technology
Runtime| Bun 1.44+
Language| TypeScript
Media demuxing| Mediabunny / equivalent
Video decoding| WebCodecs
Audio decoding| WebCodecs
Media transport| WebRTC DataChannel
Session signaling| WebSocket
Connectivity| STUN / TURN
Local file access| File API
Buffering| Application-managed bounded queues
Rendering| Canvas / WebGL
Playback coordination| WebRTC DataChannel / WebSocket

These are initial implementation candidates, not mandatory dependencies. Each should be validated against the actual media pipeline before adoption.

9. Non-goals for the initial version

- Media library management.
- Permanent cloud storage.
- Media hosting or public streaming.
- User accounts and social features.
- Multiple simultaneous viewers.
- Video transcoding.
- Adaptive bitrate renditions.
- Video editing.
- General-purpose media server functionality.
- Cross-browser support beyond the initial Chrome target.

10. Initial acceptance criteria

The first working version must demonstrate:

1. Host selects a local 1080p HEVC file.
2. Host creates a session and shares a link.
3. Recipient joins using latest stable Chrome.
4. Both devices establish a peer-to-peer connection.
5. Original HEVC encoded samples are transferred without transcoding.
6. Recipient decodes and displays the video.
7. Audio and video remain synchronized.
8. Host play, pause and seek operations propagate to the recipient.
9. Both sender and receiver adapt their buffering to changing network conditions.
10. Temporary bandwidth reductions do not cause unbounded memory growth.
11. Playback recovers gracefully after temporary network interruptions.
12. Different recipient screen dimensions do not alter the transmitted source media.

11. Fundamental invariants

- The original file is the source of truth.
- No sender-side video decoding or re-encoding is required for the passthrough path.
- Encoded media samples must retain their decoding and presentation timing information.
- Playback control belongs to the host.
- The receiver must never independently advance beyond available media.
- Buffering must remain bounded on both devices.
- Network throughput must not be confused with playback time.
- Display scaling and cropping must not modify the source media.
- Unsupported codec configurations must be detected rather than silently worked around.
- Media transfer, playback synchronization and rendering must remain separate responsibilities.

---

Primary objective: Deliver the experience of watching the same locally owned HEVC video together, with the host controlling playback and the recipient receiving the original media in real time, without transcoding or unnecessary infrastructure.
