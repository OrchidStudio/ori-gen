# V2App.jsx Refactoring Plan

**Document Version:** 1.0
**Created:** 2026-01-07
**Status:** Ready for Implementation
**Repository:** OrchidStudio/orchid
**Target File:** `frontend/src/v2/V2App.jsx`

---

## Executive Summary

This document outlines a comprehensive plan to refactor `V2App.jsx` - a 2,533-line "god component" - into a maintainable, type-safe TypeScript architecture. The plan was developed through multi-model AI consensus (Gemini 3 Pro Preview, Grok 4.1, Claude Opus 4.5, Codex GPT5.2) and includes 33 tasks with 136 TDD test specifications.

---

## Table of Contents

1. [Current State Analysis](#current-state-analysis)
2. [Multi-Model Consensus](#multi-model-consensus)
3. [Target Architecture](#target-architecture)
4. [Task Breakdown (33 Tasks)](#task-breakdown)
5. [TDD Test Plan (136 Tests)](#tdd-test-plan)
6. [Key TypeScript Interfaces](#key-typescript-interfaces)
7. [Risk Mitigation](#risk-mitigation)
8. [Implementation Timeline](#implementation-timeline)

---

## Current State Analysis

### File Metrics

| Metric | Value |
|--------|-------|
| Total Lines | 2,533 |
| Imports | 60+ |
| Context Providers | 6 nested |
| Reducers | 2 |
| useState hooks | 15+ |
| useRef hooks | 10+ |
| useEffect blocks | 8+ |
| Key functions | 8 |

### Current Concerns Handled by V2App.jsx

| Concern | What it does |
|---------|--------------|
| Recording State | Start/stop/pause via `recordingReducer` |
| Transcript Processing | Buffering, Qdrant ingestion, speaker remapping |
| Device Management | Enumeration, selection, hot-swap during recording |
| Data Loading | Questions + beat sheets from Qdrant (session + global) |
| Context Providers | Wraps 6+ context providers |
| Modal Management | Theme settings, login, question upload modals |
| Auth Integration | Login/logout flows |
| HMR State Preservation | Dev-time state persistence |

---

## Multi-Model Consensus

### Universal Agreement (4/4 models)

1. **Feature-based decomposition** into `features/` directory structure
2. **Extract pure state first** - reducers and types are lowest risk
3. **TypeScript discriminated unions** for state machines (`'IDLE' | 'RECORDING' | 'PAUSED'` not booleans)
4. **Preserve useRef patterns** for async callbacks to avoid stale closures
5. **Event-driven decoupling** between recording ↔ transcription
6. **Incremental migration** with feature flags for safe rollback

### Nuanced Perspectives

| Model | Unique Insight |
|-------|---------------|
| Gemini 3 Pro Preview | "Coordinator Pattern" - use `AppOrchestrator.tsx` to wire features via reactive effects |
| Grok 4.1 | "Strangler Fig" - keep god component as shell, replace internals incrementally |
| Claude Opus 4.5 | Event bus for cross-feature communication; explicit provider dependency graph |
| Codex GPT5.2 | Back-pressure risk in transcript queue; add max depth/retry policy |

### Critical Risks Identified

1. **Stale Closures** - `useRefSync` usage must be preserved in extracted hooks
2. **HMR State Loss** - `RESTORE_STATE` action must survive refactor
3. **Race Conditions** - `startRecordingInFlightRef` guard must stay in `finally` block
4. **Transcript Queue** - No back-pressure; can grow unbounded on ingestion failures
5. **Tauri Listener Cleanup** - Promises return unlisten functions; easy to leak

---

## Target Architecture

```
frontend/src/v2/
├── features/
│   ├── recording/
│   │   ├── index.ts
│   │   ├── RecordingContext.tsx
│   │   ├── useRecording.ts
│   │   ├── useRecordingControls.ts
│   │   ├── recordingReducer.ts
│   │   └── types.ts
│   ├── transcription/
│   │   ├── index.ts
│   │   ├── TranscriptionContext.tsx
│   │   ├── useTranscription.ts
│   │   ├── useTranscriptListener.ts
│   │   ├── transcriptReducer.ts
│   │   ├── transcriptBuffer.ts
│   │   └── types.ts
│   ├── devices/
│   │   ├── index.ts
│   │   ├── useDeviceManager.ts
│   │   ├── useDeviceHotSwap.ts
│   │   └── types.ts
│   ├── content/
│   │   ├── index.ts
│   │   ├── useQuestionsLoader.ts
│   │   ├── useBeatSheetsLoader.ts
│   │   └── types.ts
│   └── session/
│       ├── index.ts
│       └── useSessionManager.ts
├── hooks/
│   ├── useHMRStatePreservation.ts
│   └── useSpeakerMapping.ts
├── types/
│   └── domain.ts
└── V2App.tsx (~200 lines, slim orchestrator)
```

---

## Task Breakdown

### Phase 1: Foundation (XS tasks)

| # | Size | Task | Risk | Deps |
|---|------|------|------|------|
| 1 | XS | Create `types/domain.ts` with `RecordingStatus`, `RecordingState`, `RecordingAction` types | LOW | None |
| 2 | XS | Create `types/domain.ts` with `TranscriptSegment`, `TranscriptState`, `TranscriptAction` types | LOW | None |
| 3 | XS | Create `types/domain.ts` with `AudioDevice`, `DeviceState` types | LOW | None |
| 4 | XS | Create `types/domain.ts` with `SessionInfo`, `BufferWindow` types | LOW | None |
| 5 | XS | Extract `recordingReducer.ts` + add discriminated union action types | LOW | #1 |
| 6 | XS | Extract `transcriptReducer.ts` + add discriminated union action types | LOW | #2 |
| 7 | XS | Write unit tests for `recordingReducer` (all state transitions) | LOW | #5 |
| 8 | XS | Write unit tests for `transcriptReducer` (all state transitions) | LOW | #6 |

### Phase 2: Isolated Hooks (XS-S tasks)

| # | Size | Task | Risk | Deps |
|---|------|------|------|------|
| 9 | XS | Extract `useHMRStatePreservation.ts` (dev-only hook) | LOW | None |
| 10 | XS | Extract `useSpeakerMapping.ts` (just refs, no other hooks) | LOW | #3 |
| 11 | XS | Extract `useAudioServerErrors.ts` (just Tauri listener) | LOW | None |
| 12 | S | Extract `transcriptBuffer.ts` from inline buffer helpers | LOW | #2 |
| 13 | S | Add backpressure + max depth to transcript buffer | MED | #12 |

### Phase 3: Device Management (S tasks)

| # | Size | Task | Risk | Deps |
|---|------|------|------|------|
| 14 | S | Create `useDeviceManager.ts` (combines enumeration + selection) | MED | #3 |
| 15 | S | Create `useDeviceHotSwap.ts` (hot-swap during recording) | MED | #14, #5 |
| 16 | XS | Write integration test for device hot-swap flow | MED | #15 |

### Phase 4: Transcription Pipeline (S tasks)

| # | Size | Task | Risk | Deps |
|---|------|------|------|------|
| 17 | S | Create `transcriptEvents.ts` - typed Tauri listener adapter | MED | #2, #12 |
| 18 | S | Create `useTranscriptListener.ts` - consumes adapter, feeds reducer | MED | #6, #17 |
| 19 | S | Create `useTranscription.ts` - combines reducer + listener + buffer | MED | #18 |
| 20 | S | Create `TranscriptionContext.tsx` - wraps hook for sharing | MED | #19 |
| 21 | XS | Write integration test for transcript ingestion flow | MED | #20 |

### Phase 5: Recording Core (S tasks)

| # | Size | Task | Risk | Deps |
|---|------|------|------|------|
| 22 | S | Create `recordingService.ts` - wraps Tauri invoke/listen for start/stop | HIGH | #5 |
| 23 | S | Create `useRecordingControls.ts` - start/stop/pause/resume | HIGH | #22 |
| 24 | S | Create `useRecording.ts` - combines reducer + controls + session | HIGH | #23 |
| 25 | S | Create `RecordingContext.tsx` - wraps hook for sharing | HIGH | #24 |
| 26 | S | Write integration test for full recording lifecycle | HIGH | #25 |

### Phase 6: Content Loading (S tasks)

| # | Size | Task | Risk | Deps |
|---|------|------|------|------|
| 27 | S | Create `useQuestionsLoader.ts` - refreshQuestions logic | MED | #4 |
| 28 | S | Create `useBeatSheetsLoader.ts` - refreshBeatSheets logic | MED | #4 |
| 29 | S | Create `useSessionManager.ts` - session creation/sync | MED | #4 |

### Phase 7: Orchestration (S tasks)

| # | Size | Task | Risk | Deps |
|---|------|------|------|------|
| 30 | S | Create `AppOrchestrator.tsx` - wires recording ↔ transcription events | MED | #20, #25 |
| 31 | S | Refactor `V2App.tsx` to slim orchestrator (~200 lines) | MED | #30 |
| 32 | S | Add feature flag toggle between old/new architecture | LOW | #31 |
| 33 | XS | Write E2E test for full recording → transcript → stop flow | HIGH | #31 |

### Summary

| Phase | Tasks | Sizes | Risk |
|-------|-------|-------|------|
| 1. Foundation | 8 | 8×XS | LOW |
| 2. Isolated Hooks | 5 | 3×XS, 2×S | LOW-MED |
| 3. Device Management | 3 | 1×XS, 2×S | MED |
| 4. Transcription | 5 | 1×XS, 4×S | MED |
| 5. Recording Core | 5 | 5×S | HIGH |
| 6. Content Loading | 3 | 3×S | MED |
| 7. Orchestration | 4 | 1×XS, 3×S | MED-HIGH |
| **TOTAL** | **33** | **14×XS, 19×S** | |

---

## TDD Test Plan

**Total Tests: 136**

### Phase 1: Foundation (35 tests)

#### Task #1: RecordingStatus/RecordingState/RecordingAction types
**Tests: 3**

1. **Type compilation test** - Verify `RecordingStatus` discriminated union only allows valid values (`'IDLE' | 'STARTING' | 'RECORDING' | 'PAUSED' | 'STOPPING'`)
2. **Required fields enforcement** - `RecordingState` requires `status`, `sessionId`, `startedAt`, `pausedDuration`, `devices`
3. **Action type narrowing** - Each `RecordingAction` type narrows correctly in switch statements

#### Task #2: TranscriptSegment/TranscriptState/TranscriptAction types
**Tests: 3**

1. **TranscriptSegment required fields** - `id`, `text`, `speakerId`, `type`, `startTime`, `endTime` are required
2. **Discriminated union for segment type** - Only `'interim' | 'final'` allowed
3. **TranscriptAction narrowing** - Actions like `ADD_SEGMENT`, `UPDATE_SEGMENT`, `REMOVE_SEGMENT` narrow correctly

#### Task #3: AudioDevice/DeviceState types
**Tests: 3**

1. **AudioDevice structure** - Has `id`, `name`, `kind` (`'input' | 'output'`), `isDefault`
2. **DeviceState required fields** - `inputDevices`, `outputDevices`, `selectedInput`, `selectedOutput`
3. **Device ID type safety** - Device IDs are branded strings, not plain strings

#### Task #4: SessionInfo/BufferWindow types
**Tests: 4**

1. **SessionInfo required fields** - `id`, `projectId`, `createdAt`, `status`
2. **BufferWindow required fields** - `sessionId`, `transcripts`, `windowStartMs`, `maxDepth`
3. **BufferWindow.transcripts type** - Must be `TranscriptSegment[]`
4. **maxDepth positive integer** - Type enforces positive number

#### Task #5: recordingReducer extraction
**Tests: 5**

1. **IDLE → STARTING transition** - `START_RECORDING` action from IDLE state transitions to STARTING
2. **STARTING → RECORDING transition** - `RECORDING_STARTED` action transitions to RECORDING with sessionId
3. **RECORDING → PAUSED transition** - `PAUSE_RECORDING` action transitions to PAUSED, tracks pausedDuration
4. **RESTORE_STATE action** - HMR state restoration replaces entire state object
5. **Invalid transition blocked** - STARTING state ignores duplicate START_RECORDING

#### Task #6: transcriptReducer extraction
**Tests: 5**

1. **ADD_SEGMENT action** - Adds new segment to state with correct id
2. **Interim → Final dedup** - Adding final segment with same id replaces interim
3. **Speaker remapping** - REMAP_SPEAKER action updates all segments with old speakerId
4. **CLEAR_SEGMENTS action** - Resets segments array to empty
5. **RESTORE_STATE action** - HMR restoration works correctly

#### Task #7: recordingReducer unit tests
**Tests: 5**

1. **All state transitions covered** - Test matrix of (currentState × action) combinations
2. **Timing accuracy** - `startedAt` timestamp is set correctly on RECORDING_STARTED
3. **Pause duration accumulation** - Multiple pause/resume cycles accumulate correctly
4. **Idempotency** - Same action applied twice doesn't corrupt state
5. **Type safety** - Invalid actions rejected at compile time

#### Task #8: transcriptReducer unit tests
**Tests: 4**

1. **Segment ordering** - Segments maintain chronological order by startTime
2. **Concurrent updates** - Rapid ADD_SEGMENT calls don't lose data
3. **Large segment list performance** - 1000+ segments don't degrade performance
4. **Memory cleanup** - Old segments can be garbage collected after buffer eviction

---

### Phase 2: Isolated Hooks (19 tests)

#### Task #9: useHMRStatePreservation
**Tests: 4**

1. **State saved to sessionStorage** - On state change, serialized state is stored
2. **State restored on mount** - If sessionStorage has data, state is restored
3. **No-op in production** - Hook does nothing when `import.meta.env.DEV` is false
4. **Cleanup on unmount** - Clears sessionStorage key on unmount

#### Task #10: useSpeakerMapping
**Tests: 4**

1. **Ref identity stable** - `speakerMapRef.current` identity doesn't change across renders
2. **deviceId → speakerId lookup** - `getSpeaker(deviceId)` returns correct speakerId
3. **Unregistered device handling** - Unknown deviceId returns fallback speakerId
4. **Update speaker mapping** - `setSpeaker(deviceId, speakerId)` updates the map

#### Task #11: useAudioServerErrors
**Tests: 3**

1. **Tauri listener setup** - On mount, registers listener for `audio-server-error` event
2. **Error callback invoked** - When event fires, callback receives error payload
3. **Cleanup on unmount** - Unlisten function called on component unmount

#### Task #12: transcriptBuffer extraction
**Tests: 4**

1. **Add segment to buffer** - `buffer.add(segment)` increases buffer length
2. **Time-window eviction** - Segments older than `windowDuration` are evicted
3. **Get segments in range** - `buffer.getRange(startMs, endMs)` returns correct subset
4. **Buffer clear** - `buffer.clear()` resets to empty state

#### Task #13: transcriptBuffer backpressure
**Tests: 5**

1. **maxDepth enforcement** - Buffer rejects new segments when at maxDepth
2. **Oldest eviction on overflow** - When maxDepth reached, oldest segment is evicted
3. **Recovery after eviction** - After eviction, new segments can be added
4. **Backpressure callback** - `onBackpressure` callback invoked when limit hit
5. **Depth configuration** - maxDepth can be configured at creation

---

### Phase 3: Device Management (12 tests)

#### Task #14: useDeviceManager
**Tests: 5**

1. **Enumerate input devices** - Returns list of available input devices
2. **Enumerate output devices** - Returns list of available output devices
3. **Auto-select default device** - Default device is pre-selected on mount
4. **Select device manually** - `selectInput(deviceId)` updates selectedInput
5. **Error state on enumeration failure** - Sets error state if Tauri call fails

#### Task #15: useDeviceHotSwap
**Tests: 4**

1. **Disconnect detection** - Detects when selected device is disconnected
2. **Auto-pause on disconnect** - Recording pauses when input device disconnected
3. **Resume with new device** - After selecting new device, recording can resume
4. **No action if not recording** - Disconnect while idle doesn't trigger pause

#### Task #16: Device hot-swap integration test
**Tests: 3**

1. **No transcript loss during swap** - Transcripts received before disconnect are preserved
2. **Session metadata updated** - Session records device change event
3. **UI reflects new device** - Device selector shows new device as selected

---

### Phase 4: Transcription Pipeline (19 tests)

#### Task #17: transcriptEvents adapter
**Tests: 4**

1. **Tauri event parsing** - Converts raw Tauri event to typed `TranscriptEvent`
2. **Malformed payload handling** - Invalid payload returns error result
3. **Event type discrimination** - Correctly identifies `interim` vs `final` events
4. **Metadata extraction** - Extracts deviceId, timestamp, confidence from payload

#### Task #18: useTranscriptListener
**Tests: 4**

1. **Debounce rapid events** - Events within 16ms are batched
2. **Batch dispatch to reducer** - Batched events dispatched as single action
3. **Listener cleanup** - Unlisten called on unmount
4. **Error boundary integration** - Errors in listener don't crash app

#### Task #19: useTranscription
**Tests: 5**

1. **Qdrant ingestion trigger** - When buffer reaches threshold, ingest is called
2. **Retry on ingestion failure** - Failed ingestion is retried with backoff
3. **Loading state exposed** - `isIngesting` boolean reflects ingestion status
4. **Segments accessible** - `segments` array contains current transcript data
5. **Manual refresh** - `refresh()` function triggers re-fetch from Qdrant

#### Task #20: TranscriptionContext
**Tests: 3**

1. **Provider renders children** - Children render inside provider
2. **Context throws outside provider** - `useTranscriptionContext()` throws if no provider
3. **Context value stable** - Context value reference is stable across renders

#### Task #21: Transcript ingestion integration
**Tests: 3**

1. **Auto-ingest at threshold** - When 50 segments buffered, ingestion starts
2. **Ingested segments marked** - After ingestion, segments have `ingested: true`
3. **No duplicate ingestion** - Already-ingested segments not sent again

---

### Phase 5: Recording Core (21 tests)

#### Task #22: recordingService
**Tests: 5**

1. **start() invokes Tauri** - `invoke('start_recording', params)` called
2. **stop() invokes Tauri** - `invoke('stop_recording')` called
3. **Event listener setup** - Listeners for `recording-started`, `recording-stopped` registered
4. **Error propagation** - Tauri errors propagate as rejected promises
5. **Cleanup function returned** - Service exposes cleanup for unmount

#### Task #23: useRecordingControls
**Tests: 5**

1. **startRecording sets inflight guard** - `startRecordingInFlightRef` set to true
2. **Inflight guard in finally block** - Guard cleared in finally, not just on success
3. **Error rollback** - On start failure, state returns to IDLE
4. **Pause/resume cycle** - Can pause and resume multiple times
5. **Stop clears session** - stopRecording clears sessionId from state

#### Task #24: useRecording
**Tests: 4**

1. **State exposed** - `recordingState` reflects current reducer state
2. **Controls exposed** - `start`, `stop`, `pause`, `resume` functions available
3. **Session sync** - New session created on start, persisted to Qdrant
4. **Reactivity** - State changes trigger re-render in consuming components

#### Task #25: RecordingContext
**Tests: 3**

1. **Provider renders children** - Children render inside provider
2. **Context throws outside provider** - `useRecordingContext()` throws if no provider
3. **Context value stable** - Context value reference is stable across renders

#### Task #26: Recording lifecycle integration
**Tests: 4**

1. **Full cycle test** - IDLE → start → RECORDING → stop → IDLE
2. **Pause/resume within cycle** - RECORDING → pause → PAUSED → resume → RECORDING → stop
3. **Crash recovery** - If app crashes during RECORDING, can recover state on reload
4. **Concurrent start prevention** - Second start() call while STARTING is ignored

---

### Phase 6: Content Loading (13 tests)

#### Task #27: useQuestionsLoader
**Tests: 5**

1. **Load from Qdrant** - Fetches questions from Qdrant on mount
2. **Session + global merge** - Combines session-specific and global questions
3. **Refresh function** - `refresh()` re-fetches from Qdrant
4. **Loading state** - `isLoading` true during fetch
5. **Error handling** - `error` state set on fetch failure

#### Task #28: useBeatSheetsLoader
**Tests: 4**

1. **Load beat sheets** - Fetches beat sheets on mount
2. **Filter by projectId** - Only returns beat sheets for current project
3. **Cache results** - Subsequent calls return cached data
4. **Cache invalidation** - `refresh()` bypasses cache

#### Task #29: useSessionManager
**Tests: 4**

1. **Auto-create on recording start** - New session created when recording begins
2. **Session sync to Qdrant** - Session metadata persisted to Qdrant
3. **Session recovery** - Can recover incomplete session on app reload
4. **Session end timestamp** - `endedAt` set when recording stops

---

### Phase 7: Orchestration (17 tests)

#### Task #30: AppOrchestrator
**Tests: 4**

1. **Recording → Transcription wiring** - Recording start triggers transcription listener setup
2. **Transcription → Recording wiring** - Stop recording triggers final transcript flush
3. **Error boundary** - Errors in one feature don't crash the other
4. **Event ordering** - Events processed in correct order

#### Task #31: V2App slim refactor
**Tests: 4**

1. **Line count under 200** - Refactored V2App.tsx is ≤200 lines
2. **No inline business logic** - All logic delegated to feature modules
3. **Provider composition correct** - All required providers are present and ordered correctly
4. **Backward compatibility** - Old test suite still passes

#### Task #32: Feature flag
**Tests: 4**

1. **Toggle old architecture** - `FEATURE_FLAG_V2_REFACTOR=false` uses old code
2. **Toggle new architecture** - `FEATURE_FLAG_V2_REFACTOR=true` uses new code
3. **Read from environment** - Flag value read from `import.meta.env`
4. **Log mode on startup** - Console logs which architecture is active

#### Task #33: E2E test
**Tests: 5**

1. **Full recording flow** - User can start recording, see transcripts, stop
2. **Device disconnect during recording** - App handles gracefully, can continue
3. **Page refresh survival** - Recording state survives page refresh (HMR)
4. **Multiple sessions** - Can complete multiple recording sessions
5. **No memory leaks** - Memory usage stable across multiple sessions

---

## Key TypeScript Interfaces

```typescript
// =============================================================================
// Recording Domain Types
// =============================================================================

type RecordingStatus = 'IDLE' | 'STARTING' | 'RECORDING' | 'PAUSED' | 'STOPPING';

interface RecordingState {
  status: RecordingStatus;
  sessionId: string | null;
  startedAt: number | null;
  pausedDuration: number;
  devices: AudioDevice[];
}

type RecordingAction =
  | { type: 'START_RECORDING' }
  | { type: 'RECORDING_STARTED'; sessionId: string; startedAt: number }
  | { type: 'PAUSE_RECORDING' }
  | { type: 'RESUME_RECORDING' }
  | { type: 'STOP_RECORDING' }
  | { type: 'RECORDING_STOPPED' }
  | { type: 'SET_DEVICES'; devices: AudioDevice[] }
  | { type: 'RESTORE_STATE'; state: RecordingState };

// =============================================================================
// Transcription Domain Types
// =============================================================================

interface TranscriptSegment {
  id: string;
  text: string;
  speakerId: string;
  deviceId?: string;
  type: 'interim' | 'final';
  startTime: number;
  endTime: number;
  confidence?: number;
  ingested?: boolean;
}

interface TranscriptState {
  segments: TranscriptSegment[];
  isIngesting: boolean;
  lastIngestedAt: number | null;
  error: Error | null;
}

type TranscriptAction =
  | { type: 'ADD_SEGMENT'; segment: TranscriptSegment }
  | { type: 'UPDATE_SEGMENT'; id: string; updates: Partial<TranscriptSegment> }
  | { type: 'REMOVE_SEGMENT'; id: string }
  | { type: 'REMAP_SPEAKER'; oldSpeakerId: string; newSpeakerId: string }
  | { type: 'MARK_INGESTED'; ids: string[] }
  | { type: 'SET_INGESTING'; isIngesting: boolean }
  | { type: 'SET_ERROR'; error: Error | null }
  | { type: 'CLEAR_SEGMENTS' }
  | { type: 'RESTORE_STATE'; state: TranscriptState };

// =============================================================================
// Device Domain Types
// =============================================================================

interface AudioDevice {
  id: string;
  name: string;
  kind: 'input' | 'output';
  isDefault: boolean;
}

interface DeviceState {
  inputDevices: AudioDevice[];
  outputDevices: AudioDevice[];
  selectedInput: string | null;
  selectedOutput: string | null;
  isEnumerating: boolean;
  error: Error | null;
}

// =============================================================================
// Session Domain Types
// =============================================================================

interface SessionInfo {
  id: string;
  projectId: string | null;
  createdAt: number;
  endedAt: number | null;
  status: 'active' | 'completed' | 'interrupted';
  deviceHistory: Array<{ deviceId: string; timestamp: number }>;
}

interface BufferWindow {
  sessionId: string;
  projectId?: string;
  transcripts: TranscriptSegment[];
  windowStartMs: number;
  maxDepth: number;
}
```

---

## Risk Mitigation

### 1. Stale Closures
**Risk:** Extracted hooks may have stale closure references
**Mitigation:** Preserve `useRefSync` pattern in all extracted hooks; add tests for callback identity

### 2. HMR State Loss
**Risk:** State restoration may break during refactor
**Mitigation:** Task #9 extracts HMR preservation first; E2E test (#33) validates survival

### 3. Race Conditions
**Risk:** Recording start/stop race conditions
**Mitigation:** Task #23 explicitly tests inflight guard in finally block; integration test covers

### 4. Transcript Queue Unbounded Growth
**Risk:** Ingestion failures cause memory growth
**Mitigation:** Task #13 adds backpressure with max depth; test validates eviction

### 5. Tauri Listener Leaks
**Risk:** Unlisten functions not called on unmount
**Mitigation:** Every listener hook has cleanup test; ESLint rule for missing cleanup

---

## Implementation Timeline

| Phase | Tasks | Size |
|-------|-------|------|
| 1. Foundation | 8 | 8×XS |
| 2. Isolated Hooks | 5 | 3×XS, 2×S |
| 3. Device Management | 3 | 1×XS, 2×S |
| 4. Transcription | 5 | 1×XS, 4×S |
| 5. Recording Core | 5 | 5×S |
| 6. Content Loading | 3 | 3×S |
| 7. Orchestration | 4 | 1×XS, 3×S |
| **TOTAL** | **33** | **14×XS, 19×S** |

---

## Appendix: Task Dependency Graph

```
Phase 1 (Foundation)
├── #1 Recording types ─────────────────────────────┐
├── #2 Transcript types ────────────────────────────┤
├── #3 Device types ────────────────────────────────┤
├── #4 Session types ───────────────────────────────┤
│                                                   │
├── #5 recordingReducer ──── depends on #1 ─────────┤
├── #6 transcriptReducer ─── depends on #2 ─────────┤
├── #7 recording tests ───── depends on #5 ─────────┤
└── #8 transcript tests ──── depends on #6 ─────────┘

Phase 2 (Isolated Hooks)
├── #9 useHMRStatePreservation (no deps)
├── #10 useSpeakerMapping ──── depends on #3
├── #11 useAudioServerErrors (no deps)
├── #12 transcriptBuffer ───── depends on #2
└── #13 buffer backpressure ── depends on #12

Phase 3 (Device Management)
├── #14 useDeviceManager ───── depends on #3
├── #15 useDeviceHotSwap ───── depends on #14, #5
└── #16 hot-swap tests ─────── depends on #15

Phase 4 (Transcription)
├── #17 transcriptEvents ───── depends on #2, #12
├── #18 useTranscriptListener ─ depends on #6, #17
├── #19 useTranscription ────── depends on #18
├── #20 TranscriptionContext ── depends on #19
└── #21 ingestion tests ─────── depends on #20

Phase 5 (Recording Core)
├── #22 recordingService ────── depends on #5
├── #23 useRecordingControls ── depends on #22
├── #24 useRecording ────────── depends on #23
├── #25 RecordingContext ────── depends on #24
└── #26 lifecycle tests ─────── depends on #25

Phase 6 (Content Loading)
├── #27 useQuestionsLoader ──── depends on #4
├── #28 useBeatSheetsLoader ─── depends on #4
└── #29 useSessionManager ───── depends on #4

Phase 7 (Orchestration)
├── #30 AppOrchestrator ─────── depends on #20, #25
├── #31 V2App slim refactor ─── depends on #30
├── #32 feature flag ────────── depends on #31
└── #33 E2E test ────────────── depends on #31
```

---

*Document generated by ORI (Orchid Repository Intelligence) via multi-model consensus on 2026-01-07*
