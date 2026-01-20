# Video Playback Enhancement Plan

Date: 2026-01-20

## Overview
This plan delivers a two-phase upgrade to the video playback experience:

- **Phase 1 (Executable)**: Align **Short Playback** streaming to the **current External Playback** streaming behavior and session lifecycle (Option A).
- **Phase 2 (Planned)**: Reduce streaming startup latency to **time-to-first-frame (TTFF) < 5 seconds** and improve overall streaming responsiveness.

The plan is scoped to the existing Video services and their documented behavior. No code changes are included in this document.

## Goals
- **Phase 1**: Short Playback uses the same streaming mechanics as External Playback.
- **Phase 2**: Streaming initiates with TTFF < 5 seconds for supported VMS backends.
- **Optional**: Add observability to measure and track startup latency.

## Non-goals
- Major UI redesigns or new player features.
- Migrating to a different streaming protocol unless required by Phase 2 findings.

## Definitions
- **Short Playback**: Automatic case footage currently delivered as pre-generated MP4/JPEG.
- **External Playback**: User-initiated historical playback using WebSocket streaming.
- **TTFF**: Time-to-first-frame from client request to first renderable frame on the UI.

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
   - Add references in [README.md](../README.md) to this plan.

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
**Objective**: Reduce streaming startup latency to **TTFF < 5 seconds** and improve overall perceived responsiveness.

### Plan Items
1. **Measure Baseline TTFF**
   - Capture current TTFF for each VMS path.
   - Use consistent measurement from request initiation to first UI frame.

2. **Streaming Pipeline Optimization**
   - Identify startup bottlenecks (auth, session creation, RTSP retrieval, FFmpeg spin-up, frame extraction).
   - Consider reusing hot sessions where safe and feasible.

3. **Pre-initialization Opportunities**
   - Pre-auth or prefetch camera session tokens for frequently accessed cameras.
   - Cache configuration and endpoint discovery results where valid.

4. **Transport and Frame Handling**
   - Evaluate frame batching vs single-frame delivery at startup.
   - Reduce first-frame latency by prioritizing immediate keyframe retrieval.

5. **Observability (Nice-to-have)**
   - Add TTFF metrics and structured logs per playback session.
   - Optional tracing for request-to-first-frame spans.

### Considerations
- Maintain consistent behavior across VMS backends while allowing vendor-specific optimizations.
- Ensure improvements do not regress playback stability or frame quality.
- Document vendor-specific limitations that prevent TTFF < 5 seconds.

### Phase 2 Success Criteria (Target)
- TTFF < 5 seconds for supported vendors and supported network conditions.
- Clear telemetry showing startup latency distribution and regressions.

---

## Risks and Mitigations
- **Risk**: Short Playback streaming causes increased load.
  - **Mitigation**: Enforce time-window limits and introduce rate limiting.
- **Risk**: VMS-specific startup variability.
  - **Mitigation**: Vendor-specific fallbacks and explicit SLA documentation.

## Documentation Updates Required
- [docs/video-feature-documentation.md](video-feature-documentation.md)
- [README.md](../README.md)
- [CHANGELOG.md](../CHANGELOG.md)
