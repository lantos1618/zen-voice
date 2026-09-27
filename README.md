# zen-voice

A headless Zen voice pipeline: actor-backed Parakeet inference, a polled result
mailbox, and bounded live dictation admission. There are no AppKit, microphone,
window, FFT, or display-history dependencies. An application supplies PCM and
owns capture, session IDs, error presentation, and committed text.

## Import and build

Register source libraries in the executable's build graph:

```zen
parakeet = b.lib("parakeet", {
    src: Path("../zen-parakeet/src/parakeet.zen"),
    libs: ["nemo_speech_asr_c"],
    paths: ["../zen-parakeet/build/nemo-speech/lib"]
}).try();
voice = b.lib("voice", {
    src: Path("../zen-voice/src/voice.zen"), libs: [], paths: []
}).try();
// Include both parakeet and voice in b.exe's deps.
```

```zen
Transcriber, Transcript, MailboxError, Live, quiet_tail = voice
```

The native NeMo SDK headers and library search/rpath must be configured as in
zen-parakeet. This repository does not install a model or download dependencies.
`tests/run.py` supplies a complete headless executable build without a UI SDK.

## Ownership and messages

`Transcriber.open(env, alloc)` starts an inference actor and a result actor.
Keep the allocator alive through `close()`, use one owning handle, and do not
copy a live handle. Actor execution uses Zen's pthread-backed actor runtime.
The caller may poll from its own event loop; actors never call application UI.

`submit(model, gpu, samples, count, rate)` accepts one offline job.
`snapshot(model, gpu, session, samples, count, rate, finish)` attaches a nonzero
capture session and caps each request at 480,000 float32 samples. `segment`
accepts separate `finalize` and `ends_capture` flags: a finalized phrase need
not end the capture. All return `Ok(false)` when admission is refused, including
when already busy. There is one outstanding request and no queued audio backlog.
The model path and backend are pinned at the first accepted request. Later
requests with a different path or backend return `Ok(false)`; close and reopen
to change identity, including after a failed load. Paths are compared byte for
byte, limited to 4,096 bytes, and must not contain NUL. CPU selection is `gpu = -1`.

`prepare(model, gpu)` uses the same one-job admission slot and loads the model
then decodes one second of silence on the worker. Call it before enabling
capture to move model initialization and first inference away from first speech.
The reply has `preparation = true`, session/count zero, and `final` and
`ends_capture` false. On success its text is empty (discarded warm-up output),
so it must never be committed to transcript history. On failure its text is the
error. A successful preparation warms that duration only; longer audio shapes
may still incur first-use work. Existing audio submits remain compatible and
load lazily when preparation is omitted.

The sample pointer must address `count` readable float32 values during the call.
Actor message construction copies the PCM bytes before returning. The worker
then copies those bytes to aligned float32 scratch storage before invoking the
native library. Caller buffers and native recognizer pointers are never sent
across threads as borrowed actor-message payloads.

`poll()` returns `Ok(None)` while pending or `Ok(Ok(Transcript))` when complete.
It does not wait for inference. A reply contains `preparation`, `success`, `text`, `session`,
`final`, `ends_capture`, and the exact submitted `count`. Text borrows the
mailbox buffer until the next submit/poll; copy or consume it before then.
Replies are UTF-8 truncated to 4,000 bytes. Capture audio is never implicitly
consumed by this library. Only discard a prefix after a successful current-session
finalized result has been committed by the application. Preserve any newer tail.

`close()` waits for accepted native inference, stops/joins both actors, and
closes the pipe. It is idempotent on the same handle. There is no inference
cancellation; close can take as long as the current decode.

## Live admission

`Live` tracks a capture session, last admitted prefix, completion, and failure.
Observe the current capture session before handling any reply. `current` rejects
old-session results; `committed` also refuses stale completion state changes.
`due_segment` permits a new partial after at least 16,000 additional samples
when idle. `finalize` chooses a quiet boundary after 128,000 samples or forces
one at 240,000; `segment_count` caps the submitted prefix at that hard boundary.
`quiet_tail` tests the last 4,800 samples against mean-square energy 1e-5.
These thresholds assume 16 kHz mono audio: one-second revisions, an eight-second
minimum phrase, a fifteen-second hard boundary, and a 300 ms quiet tail.

After admission call `accepted_segment`. After successfully committing and
removing exactly the reply's prefix, call `committed(session, remaining, active)`.
Stopping while busy leaves the remaining tail eligible for a final decode. A
new session resets admission and invalidates the previous reply. `fail()` blocks
further admission until a new session is observed. Callers must enforce their
capture capacity and surface overflow; this library cannot stop a microphone.

The current Parakeet TDT v3 model is offline-only. Partial results are repeated
growing-prefix inference and may revise earlier words. Finalized segments have
no overlap; forced boundaries may reduce recognition accuracy. This is not a
stateful streaming recognizer and does not promise lossless text recognition.

## Platform boundary

The source is Zen, including direct `c.bind` declarations for POSIX pipe/poll and
Darwin errno. There is no handwritten C bridge, Python runtime, or filesystem
mailbox. The current implementation targets macOS: `__error`, poll layout, and
pipe buffering assumptions must be adapted and validated before other platforms.
Shutdown relies on one maximum 4,002-byte reply fitting the macOS pipe buffer;
this is a capacity assumption, not a claim about portable `PIPE_BUF` atomicity.

## Tests

```sh
python3 tests/run.py
python3 tests/run.py --model ../zen-parakeet/models/parakeet-tdt-0.6b-v3.q8_0.gguf --wav ../zen-parakeet/build/fixture.wav
```

The mandatory native test checks preparation failure and reply tagging, model/backend identity refusal, one simulated hour of scheduler state, model failure, one-in-flight admission,
nonblocking polling, repeated close, and live busy/final/stale scheduling.
The optional WAV must be the known 16 kHz mono float32 quick-brown-fox fixture
from zen-parakeet. It first prepares the real model, rejects changed model/backend requests, then asserts recognized words in a live partial, the complete
final result, and a second finalized segment using the cached recognizer.
Tests build temporary headless executables and never record a microphone or
restart an app. ZenCode's `tests/transcription` separately checks 77.5 seconds
of sample accounting and the application's bounded UTF-8 display history.
