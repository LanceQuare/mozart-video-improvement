# Video Playback Enhancement Plan

Date: 2026-01-27

## Overview
This plan delivers a two-phase upgrade to the video playback experience:

- **Phase 1 (Executable)**: Align **Short Playback** streaming to the **current External Playback** streaming behavior and session lifecycle (Option A).
- **Phase 2 (Planned)**: Reduce streaming startup latency to **time-to-first-frame (TTFF) < 3 seconds** (server-side first decoded frame) and improve overall streaming responsiveness, with **Dahua** as the highest priority.

The plan is scoped to the existing Video services and their documented behavior. No code changes are included in this document.

## Goals
- **Phase 1**: Short Playback uses the same streaming mechanics as External Playback.
- **Phase 2**: Streaming initiates with TTFF < 3 seconds for Dahua (live view and external playback).
- **Observability**: Log-only TTFF measurement at debug level with session ID, camera ID, and stream type.

## Non-goals
- Major UI redesigns or new player features.
- Migrating to a different streaming protocol unless required by Phase 2 findings.

## Definitions
- **Short Playback**: Automatic case footage currently delivered as pre-generated MP4/JPEG.
- **External Playback**: User-initiated historical playback using WebSocket streaming.
- **TTFF (Phase 2)**: Time-to-first-frame from request initiation to **first decoded frame on the server**. Client-side TTFF will be added after server-side validation.

## Current Behavior Summary (Reference)
- Short Playback: Pre-generated MP4/JPEG files from recorded footage.
- External Playback: WebSocket streaming from VMS integrations.

For the current behavior details, see [docs/video-feature-documentation.md](video-feature-documentation.md).

---

## Phase 1 — Executable Plan
**Objective**: Make Short Playback streaming align with the current External Playback flow, reusing its session lifecycle, endpoints, and playback pipeline.

### Scope
- Reuse External Playback streaming contract and session lifecycle.
- Short Playback requests initiate a WebSocket streaming session rather than retrieving MP4/JPEG.
- Maintain existing Short Playback media artifacts as fallback only if streaming fails.

### Deliverables
- Unified Short Playback streaming contract aligned with External Playback.
- Updated documentation and API contracts.
- Test coverage for Short Playback streaming paths.

### Work Breakdown
1. **Contract Alignment**
   - Document Short Playback request inputs and map them to existing External Playback request fields.
   - Define how short playback duration (e.g., 10 seconds) is represented in the request.
   - Confirm stop/pause semantics follow External Playback behavior.

2. **Backend Streaming Entry Points**
   - Add/adjust Short Playback API entry points to call the same internal streaming start path as External Playback.
   - Reuse existing session identifiers and authorization behavior.
   - Ensure the correct time window is passed (e.g., 5s before + 5s after event).

3. **Session Lifecycle Consistency**
   - Start, keepalive, and stop flows must follow the External Playback flow.
   - Ensure failure states return consistent errors to the client.

4. **Fallback Strategy**
   - If streaming cannot be established, fallback to existing MP4/JPEG assets.
   - Define explicit error thresholds that trigger fallback.

5. **Testing**
   - Contract tests for Short Playback start/stop.
   - Integration test for WebSocket stream initiation within the short playback time window.
   - Regression test to ensure External Playback remains unchanged.

6. **Documentation**
   - Update Short Playback description in [docs/video-feature-documentation.md](video-feature-documentation.md) to reflect streaming alignment.
   - Add references in [src/video/README.md](../src/video/README.md) to this plan.

### Acceptance Criteria
- Short Playback streaming uses the External Playback lifecycle and WebSocket frame delivery.
- Short Playback can stream the correct window around the event time.
- Fallback behavior is deterministic and documented.
- External Playback behavior is unchanged.

### Phase 1 Checklist
- [ ] Contract mapping and duration alignment documented
- [ ] Streaming entry point for Short Playback implemented
- [ ] Session lifecycle aligned with External Playback
- [ ] Fallback behavior defined and implemented
- [ ] Tests added/updated
- [ ] Documentation updated

---

## Phase 2 — Planned Improvements
**Objective**: Reduce streaming startup latency to **TTFF < 3 seconds** (server-side first decoded frame), prioritizing **Dahua** live view and external playback. Short playback is out of scope for Phase 2.

### Plan Items
1. **Measure Baseline TTFF (Log-only, Server-side)**
   - Capture current TTFF for **Dahua live view and external playback** only.
   - Measure from request initiation to **first decoded frame** on the server.
   - Log at **debug** level with **session ID, camera ID, and stream type**.

2. **Streaming Pipeline Optimization (Dahua-first)**
   - Identify startup bottlenecks: auth, RTSP retrieval, FFmpeg spin-up, first frame decode.
   - Prioritize improvements that reduce Dahua first-frame latency without changing stream stability.

3. **Low-res First Frame Strategy (Config-driven)**
   - Introduce **low-resolution first frame** for all Dahua streams.
   - The low-res settings apply to the **entire stream session** for efficiency (no mid-stream FFmpeg restart).
   - After the first decoded frame is logged, normal playback continues with the reduced settings.
   - Expose settings via configuration keys (width, fps, quality).

4. **Transport and Frame Handling**
   - Reduce startup latency by tuning FFmpeg startup parameters (probe/analysis duration, buffering) within safe bounds.
   - Maintain stable frame delivery after the first decoded frame.

5. **Observability (Logs Only)**
   - No metrics or tracing; **debug logs only** for TTFF and failure paths.
   - Log failures when no frame is decoded (standard logger).

### Considerations
- Phase 2 is Dahua-only for live view and external playback; other vendors remain unchanged.
- Low-res-first settings are applied to the entire stream session for efficiency (prevents mid-stream FFmpeg restart).
- Ensure improvements do not regress playback stability or frame quality.
- Document vendor-specific limitations that prevent TTFF < 3 seconds.

### Phase 2 Success Criteria (Target)
- **Dahua live view and external playback** reach TTFF < 3 seconds (server-side first decoded frame).
- Debug logs include session ID, camera ID, and stream type for TTFF correlation.
- Low-res first frame is applied for entire stream session (efficient; no mid-stream FFmpeg restart).

---

## Risks and Mitigations
- **Risk**: Short Playback streaming causes increased load.
  - **Mitigation**: Enforce time-window limits and introduce rate limiting.
- **Risk**: VMS-specific startup variability.
  - **Mitigation**: Vendor-specific fallbacks and explicit SLA documentation.
- **Risk**: Low-res first frame may cause brief visual downgrade.
  - **Mitigation**: Keep defaults conservative and configurable; applied to entire session for efficiency.

## Documentation Updates Required
- [docs/video-feature-documentation.md](video-feature-documentation.md)
- [src/video/README.md](../src/video/README.md)
