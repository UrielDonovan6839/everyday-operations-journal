# EU Startup Speech-to-Text API: Batch Quality Before Per-Minute Pricing

Short answer: for fintech sales-call summaries that become CRM actions, start with batch transcription and a quality gate; choose streaming only when agents must act before the call ends, because the cheapest per-minute speech-to-text API can still be the expensive choice after retries, review, and bad actions.

| Pick | Pick it when | Measure before committing | Main limit |
|---|---|---|---|
| Batch transcription | CRM actions can wait until the recording is complete | accepted actions per billed audio minute | Slower time to first action |
| Streaming transcription | An agent needs captions or prompts during the call | stable partials, finalization delay, and accepted actions | Partial text can trigger premature downstream work |
| Human review before write | A wrong CRM update has material operational risk | review minutes and correction rate | Adds queue time and labor |

This is a quality-versus-latency decision, not a price-page contest. OpenAI, Deepgram, AssemblyAI, and Google Cloud can all enter the same evaluation harness. Don't crown one from a headline rate. Feed each candidate the same EU sales-call sample, apply the same acceptance rules, and compare the whole path from audio received to CRM action accepted.

## Build the workflow scorecard

Compare outcomes at the workflow boundary. The useful denominator is not merely submitted audio minutes; it is the number of summaries and CRM actions that survive validation without a replay or manual correction. A provider quote still matters, but it belongs in one column beside billing increment, retry policy, regional processing requirements, transcript quality, and end-to-end latency. Your mileage may vary because accents, product names, overlapping speakers, and call-channel quality change the workload. A fresh test on representative, consented recordings resolves that uncertainty.

Use one scorecard for every candidate:

| Signal | Why it belongs in the gate | Example measurement |
|---|---|---|
| Transcript acceptance | Prevents low-confidence text from silently becoming a CRM fact | accepted calls / completed calls |
| Action acceptance | Tests the business output, not attractive prose | accepted actions / proposed actions |
| End-to-end latency | Captures queues and summarization after transcription | audio ready to CRM-ready |
| Retry amplification | Exposes work billed or processed more than once | submitted minutes / unique minutes |
| Review load | Makes the human queue visible | review minutes / audio hour |
| Regional evidence | Supports the team's EU data-handling review | recorded configuration and contract evidence |

Keep raw counts next to ratios. Ten rejected calls out of twenty means something very different from ten out of twenty thousand, and a dashboard that hides the denominator invites a confident but weak decision.

## Batch: let the final transcript settle

Batch is the default for post-call CRM actions because the pipeline can inspect a complete recording before it writes anything. Diagram in words: recording arrives, transcription runs, the transcript passes validation, summarization proposes actions, policy checks the structured output, and only then does the CRM adapter write. Each arrow gets a timestamp and a correlation ID. Crisp. Debuggable.

The key design choice is a two-stage gate. First, decide whether the transcript is usable: required speaker coverage exists, the text is nonempty, and the call identity matches the ingestion record. Second, validate the proposed CRM actions against a narrow schema and business rules. A pleasant summary can coexist with a dangerous action, such as an invented follow-up date, so one blended “quality” score is too vague for alerting. Keep transcript rejection and action rejection separate.

Batch is not suitable when an agent needs an in-call warning, live caption, or immediate next-best action. In that case, accept streaming's operational complexity and define exactly which partial results may reach a person. Do not let provisional text write directly to the CRM.

## Streaming: spend latency only where it matters

Streaming earns its place when waiting for the completed recording would make the feature useless. Set a latency budget for each boundary: audio arrival to partial text, partial text to visible hint, final audio to final transcript, and final transcript to durable CRM action. One end-to-end percentile cannot show which queue moved.

The catch is revision. A partial transcript is evidence in motion — useful for a screen, unsafe as a durable fact. Give partial and final events different types, permit only final events into the post-call action pipeline, and record how often displayed text changes before finalization. This makes the speed benefit observable without pretending every early token is settled.

Stick with batch when the feature promise is “accurate notes after the call.” Choose streaming when the promise explicitly depends on “during the call,” and test that promise with users rather than treating lower transport latency as proof of value.

## How can an EU startup trace a speech-to-text API into CRM?

The adapter below keeps vendor-specific transport behind one interface and emits the measurements needed for a fair comparison. It does not guess commercial endpoints or response shapes. Each provider implementation maps its documented API into this internal contract, while the evaluator and telemetry remain unchanged.

```ts
type Candidate = "openai" | "deepgram" | "assemblyai" | "google-cloud";

type Transcript = {
  text: string;
  final: boolean;
};

type Trial = {
  candidate: Candidate;
  callId: string;
  uniqueAudioSeconds: number;
  submittedAudioSeconds: number;
  startedAtMs: number;
};

interface SpeechToTextAdapter {
  transcribe(audio: Uint8Array, callId: string): Promise<Transcript>;
}

interface Metrics {
  count(name: string, value: number, labels: Record<string, string>): void;
  timing(name: string, milliseconds: number, labels: Record<string, string>): void;
}

async function runTrial(
  adapter: SpeechToTextAdapter,
  metrics: Metrics,
  trial: Trial,
  audio: Uint8Array,
): Promise<Transcript> {
  const labels = { candidate: trial.candidate };
  const transcript = await adapter.transcribe(audio, trial.callId);
  const accepted = transcript.final && transcript.text.trim().length > 0;

  metrics.timing("stt_end_to_end_ms", Date.now() - trial.startedAtMs, labels);
  metrics.count("stt_unique_audio_seconds", trial.uniqueAudioSeconds, labels);
  metrics.count("stt_submitted_audio_seconds", trial.submittedAudioSeconds, labels);
  metrics.count("stt_transcript_accepted", accepted ? 1 : 0, labels);
  metrics.count("stt_transcript_rejected", accepted ? 0 : 1, labels);

  if (!accepted) {
    throw new Error("Transcript did not pass the workflow acceptance gate");
  }

  return transcript;
}
```

Run this against a frozen evaluation set before launch, then shadow a bounded slice of production traffic with the same call IDs and no CRM writes. The frozen set makes candidate changes comparable. The shadow run reveals queueing, payload, and audio-shape differences that a fixture cannot capture. Store configuration revisions with every trial, because a model or option change creates a new candidate even when the provider name stays the same.

Alert on workflow symptoms. A rise in transcript rejection, retry amplification, or review minutes deserves attention before a monthly invoice does. Also split metrics by the dimensions that can explain a change — audio channel, language, call duration band, and configuration revision — while keeping customer identifiers out of metric labels. Logs can carry a controlled correlation ID; metrics should stay low-cardinality.

For pricing, calculate `current quoted charge / accepted CRM actions` from the same run. Record the quote date, currency, billing unit, rounding rule, and applicable region alongside the result. I'm not sure any static article can name the cheapest option for a specific startup without those current inputs; a signed quote and this workload test can.

## The boundary

No harness turns unlike commercial offers into perfectly identical services. Contract terms, available regions, language needs, retention controls, and support obligations can eliminate a candidate before performance testing. The four named APIs are candidates, not a ranking, and this note makes no claim that their current rates or capabilities are equal.

Use a hard decision rule: discard candidates that fail legal, regional, or workflow acceptance requirements; among the survivors, choose the lowest measured cost per accepted CRM action that meets the latency budget. Prefer batch if both modes pass. Switch to streaming only when measured user value depends on results arriving during the call.

That's the boundary.

## References

- https://github.com/openai/tiktoken
- https://python.langchain.com/docs/integrations/chat/openai/
