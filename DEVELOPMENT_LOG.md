# Awaaz se Kaam — Development Log

## Milestone 1 — Project setup ✅ (2026-08-21)

**What was done**
- Created dedicated project folder `AwaazSeKaam/` (Android app + backend service).
- Stack: Kotlin (via AGP 9.3.1 built-in Kotlin) + Jetpack Compose (BOM 2026.02.01, Material3), Gradle wrapper 9.6.1, compileSdk 37 / targetSdk 36 / minSdk 26.
- App icon intentionally skipped per instruction (default icon used).
- Stitch-inspired Home screen implemented: top bar (menu/title/profile), large mic button ("Give Task"), Pending/Done/Not Done stat cards, Recent Tasks list with status chips, bottom navigation (Home/Tasks/History).
- Fixed edge-to-edge status-bar overlap using `statusBarsPadding()`/`navigationBarsPadding()`.

**Environment established**
- JDK: Android Studio JBR 25 (`C:\Program Files\Android\Android Studio\jbr`)
- SDK: `C:\Users\sahil\AppData\Local\Android\Sdk`
- Device: Xiaomi 25053PC47I (`690acf71`), Android 16 (API 36)
- OpenRouter model `stealth/ox-alpha` **verified present** in public model list (420 models).
- No Python/Node on host → backend will be a zero-dependency single-file Java service (JDK HttpServer).

**Tested**
- `gradlew assembleDebug` → BUILD SUCCESSFUL
- `adb install -r` → Success
- Launched `com.awaazsekaam.app/.MainActivity` → process alive, no AndroidRuntime errors in logcat
- Screenshot verified against Stitch reference: layout, colors, chips, bottom nav all match.

**Blockers:** none.

## Milestone 2 — UI / Navigation ✅ (2026-08-22)

**What was done**
- Real screens replacing placeholders:
  - **Create Task**: mic action, editable live-transcription field, Parsed Details card (assignee, phone, description), confirmation bar ("Send this task to X?" + SMS preview), SEND SMS button disabled until required fields present. Creates a `READY_TO_SEND` task in the repository.
  - **Task Details**: lifecycle stepper (Created → SMS Sent → Waiting → Received → Analyzed), task detail rows, worker reply bubble, AI analysis card with confidence bar and matched-points/exceptions checklists.
  - **Analysis Result**: status circle + title (DONE/NOT DONE/PARTIAL/NEEDS REVIEW), AI remark, confidence progress bar, "WHY THIS RESULT?" checklist (✓ matched / ✗ exceptions), VIEW DETAILS / START NEW TASK, and Mark Done / Mark Not Done manual resolve for NEEDS_REVIEW.
- State-based navigation (`Screen` sealed class) with `rememberSaveable` restoration; back navigation wired.
- Home tabs (Home/Tasks/History) switch content; empty state text renders.
- Added Compose UI test infrastructure (`ui-test-junit4`, espresso-core).

**Tested (on device 690acf71 via `connectedDebugAndroidTest`)**
- 6/6 instrumented tests PASS:
  - home shows Give Task/stats
  - home → Create Task → fill form → SEND SMS → Task Details → back → Home
  - send button no-op when required fields missing
  - seeded DONE task → details → analysis result shows DONE
  - tab switching without crash
  - screenshot walkthrough captured (create_task.png / task_details.png / analysis_result.png)
- Screenshots visually verified against Stitch reference.

**Known issues / notes**
- MIUI blocks input injection from adb AND Espresso key events (`INJECT_EVENTS` SecurityException). Tests use Compose accessibility actions instead of `input tap`; back navigation tested via on-screen back button rather than system Back key. Manual impact: none for real users.
- AGP uninstalls app+test APKs after connected tests; screenshots extracted by re-installing manually and running instrumentation directly, pulling via `adb exec-out run-as`.

## Milestone 3 — Speech-to-Text ✅ (2026-08-22)

**What was done**
- `speech/SpeechToTextService.kt`: interface + `AndroidSpeechToTextService` wrapping Android `SpeechRecognizer` (en-IN for Hindi-English code-mixed speech), exposing StateFlows for listening state / partial text / final text / errors.
- Create Task screen: mic tap → runtime RECORD_AUDIO permission flow → live partial transcript streams into the editable field → final result commits → heuristic pre-fill + LLM extraction triggered.
- Clear error surfacing for all recognition error codes; graceful handling when recognizer unavailable.
- Fixed real-device bug reported by user ("Client-side recognition error"): OEM builds return ERROR_CLIENT when a recognizer instance is reused or when stopListening precedes first start. Solution: fresh recognizer per session, no premature stop, 300ms bind delay.

**Tested (real device, real human speech)**
- Multiple live utterances transcribed correctly via logcat evidence:
  - "Harsh Ek litre dudh lete Aana" ✓
  - "Amit 2 kg chawal lete Aana" ✓
  - "Sahil Aata Lekar Aana" ✓
- Partial results stream live into UI; final results commit.
- Error display verified (user saw clear error message during pre-fix ERROR_CLIENT).

**Device/platform notes**
- HyperOS blocks ALL programmatic permission grants (`pm grant`, `install -g`, UiAutomation grant) — RECORD_AUDIO must be granted once by tapping Allow in the system dialog. Grant survives app updates but not uninstall.
- MIUI adb installs intermittently fail with INSTALL_FAILED_USER_RESTRICTED; retry loop works.
- Instrumented TTS→mic loop test written (SpeechRecognitionTest) but not yet green due to permission-reset-on-reinstall; manual testing confirmed real functionality instead.

## Milestone 4 — LLM Task Extraction ✅ (mock mode) (2026-08-22)

**What was done**
- `backend/`: zero-dependency Java proxy (`AwaazBackend.java` + minimal JSON lib) exposing `/api/health`, `/api/extract-task`, `/api/verify-task`. Reads `OPENROUTER_API_KEY`, `OPENROUTER_MODEL` (verified slug: `stealth/ox-alpha`), `PORT`, `MOCK_LLM` from env/.env. Strict-output prompts, response validation, retry once on transient failures, clear 503 when key missing.
- `ai/BackendClient.kt` (OkHttp): typed results (`AiResult.Success/Failure`) — malformed model output or network failure can never crash the UI.
- Create Task screen: after final transcript → heuristic pre-fill instantly + async LLM extraction; missing assignee → clarification question shown (AI never invents); failure → amber "fill manually" notice.
- Fixed en-route: LaunchedEffect keyed on StateFlow object instead of value (extraction never fired); cleartext HTTP blocked (added narrow network_security_config allowing localhost/10.0.2.2 only); missing INTERNET permission (EPERM socket error).

**Tested (real device, end-to-end)**
- Spoke: "Harsh ko 1 litre dudh Le Aana" → STT transcript → POST /api/extract-task hit backend (log evidence) → structured result assignedToName=Harsh, quantity=1 litre, confidence=0.92 → form auto-filled, confirmation bar shown. Screenshot verified.
- Mock verify fixtures: DONE/PARTIAL/NOT_DONE/NEEDS_REVIEW classification 4/4 correct.
- Backend negative paths verified: missing key → clear 503 config error; malformed body → 400.

**Blockers / notes**
- LIVE OpenRouter calls pending user's OPENROUTER_API_KEY (not present on machine). Everything else runs against deterministic mock fixtures until then. Model slug stealth/ox-alpha verified available in OpenRouter catalog.
- Phone reaches backend via `adb reverse tcp:8081 tcp:8081` (USB tethering of localhost). For Wi-Fi demo, host LAN IP can be substituted in one constant.



