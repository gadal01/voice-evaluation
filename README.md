# QA console for the voice agent

One page, `http://127.0.0.1:8000/qa`, mounted in the agent's own app (`main.py` includes `qa_console/router.py`).
It reuses the batch harness in `evals/` (jiwer WER, clip library, noise recordings) rather than replacing it.

```
python main.py                       # agent + console   (needs AWS creds in .env for Bedrock / Nova Sonic)
# open /qa, paste AUTH_API_KEY into "Access key"
python -m pytest qa_console/tests -q # 8 tests, no AWS needed (use .venv-qa or your own env)
```

## The three tabs
1. **Live session** - pick a golden scenario, talk to the agent (mic) or replay a stored baseline clip (optionally mixed with
   noise at a chosen SNR). **● Record** captures caller audio, the audio the agent actually *played* (so barge-ins are truncated
   correctly), every WebSocket event with timestamps (transcripts, replies, tool calls with args/results/ms, interruptions).
   **■ Stop & save** stores it under `qa_data/sessions/<id>/`. **Evaluate** returns the metrics below.
2. **Sessions & metrics** - list, replay audio, re-evaluate, human spot check (naturalness / pronunciation), and aggregates by
   scenario and by noise condition (= WER under noise).
3. **Baseline clips** - record real human voices line by line (raw mic: no echo-cancel / NS / AGC), auto-trim, clipping and
   level checks, saved to `evals/caller_clips/` as `human:<speaker>`. The same clips replay in tab 1 and in
   `python evals/run_evals.py --voices clip:human:<speaker> --augment noisy --noise-snr 10`.

## Metrics (all computed in `scoring.py` / `llm.py` / `quality.py`)
| Metric | How it is computed |
|---|---|
| WER | jiwer, caller reference text vs the agent's own transcript of the caller (digits normalised). Reference = typed text, else the clip text, else the golden script |
| WER under noise | same WER, grouped by the session's condition (noise type @ SNR) |
| Slot accuracy | golden slots (name, phone, item, brand, symptom, service type...) vs the args/result of the real `create_service_request` call; per-slot ✓/✗ |
| Session outcome / task success | outcome derived from tool results (booked / quote_given / slots_given / declined / error) vs golden outcome; booking when the golden says decline is a hard fail |
| Turns to resolution | caller turns up to the resolving tool call |
| Intent accuracy | LLM classifier on the caller's words vs the labelled intent; falls back to a tool-trace proxy when no AWS creds |
| Tone / clarity / empathy / accuracy / policy | LLM judge (Bedrock Converse, 1-5, temperature 0) - advisory |
| TTS naturalness | UTMOS and DNSMOS if installed (`pip install torch torchaudio speechmos onnxruntime`) + the human 1-5 form |
| Latency | caller speech end -> agent audio start (energy VAD on both channels), p50 / p95 |
| False barge-in rate | agent `interrupted` with no caller speech within 1.5 s before / 0.3 s after; also reports missed barge-ins |
| Cost / session | audio seconds x tokens/sec x rate, plus judge tokens |

## Things to know before trusting the numbers
- `pricing.json` holds **placeholder rates** (`"verified": false`; the UI says so). Replace with your real Nova Sonic / judge pricing.
- Targets shown as green/red in the UI are starting values in `console.html` (`TARGETS`); set your own.
- LLM-judge and MOS scores are no-reference predictors: calibrate against 30+ human-labelled sessions (the Phase 2 kappa tooling
  in `rollout/02-...` does this) before using them as a release gate.
- WER here measures the **agent's** STT (Nova Sonic's transcript), which is what matters for slot errors. It does not run an
  independent ASR on the audio.
- The energy VAD times turns and barge-ins; loud steady noise can fool it, so false barge-in counts under noise deserve a listen.
- Golden rows live in `golden.json`; rows with booked outcomes were also appended to `evals/scenarios.json` so the same ids work for clips and the batch runner.
- `qa_data/` contains call recordings: keep it out of version control and apply your retention policy.
