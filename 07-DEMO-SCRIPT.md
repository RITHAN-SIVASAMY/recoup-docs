# Recoup — 5-Minute Pitch Video Script

| Field | Detail |
|---|---|
| Version | 2.0 |
| Length | 5:00 hard cap. Judges stop watching at 5:01. |
| Format | Screen recording with voice-over. Face optional; a face on the first and last 10 seconds helps. |
| Golden rule | **Show the system doing the thing. Never narrate a slide of the thing.** Slides only for the two moments where a diagram genuinely beats a screen. |

---

## Structure at a glance

| Time | Beat | What is on screen | The point being made |
|---|---|---|---|
| 0:00–0:25 | The leak | Dashboard: ₹ at risk, four sources | The problem is real and quantified |
| 0:25–1:00 | The wrong number | Raw vs. incremental side by side | We measure honestly — the hook |
| 1:00–1:45 | Diagnosis and restraint | Case detail: root cause, uplift, EV refusing to act | We know *who not to chase* |
| 1:45–2:30 | Bounded execution | Policy blocks, approval card, staged undo, kill switch | It cannot become a spam engine |
| 2:30–3:05 | The customer side | Phone: SMS → recovery page → test-mode payment → case flips live | It is a product, not a dashboard |
| 3:05–3:35 | Hinglish voice + promise | Call audio, transcript, escalation goes quiet | The thing nobody else has |
| 3:35–4:05 | Grounded audit | Q&A answers with citations, then refuses | It won't lie to a compliance reviewer |
| 4:05–4:35 | Break it | Chaos control: duplicates, crash, provider down | Graceful failure, proven live |
| 4:35–5:00 | The number and the ask | Headline block + repo + honest limitations | Defensible close |

---

## Full script

### 0:00–0:25 · The leak

> *"Every month, between eight and fifteen percent of the money customers try to send an Indian merchant never arrives. A bank declines. An OTP times out. A UPI Autopay renewal quietly stops. Someone abandons checkout. An invoice ages. This is Recoup — it watches all four, and it tries to win that money back without becoming a spam engine."*

**On screen:** dashboard loading with the live SSE stream, cases arriving from four sources, ₹ at risk climbing.

---

### 0:25–1:00 · The number that isn't there *(the hook — do not shorten this)*

> *"Most recovery tools would show you this: nearly five crore rupees recovered. That number isn't fabricated — it's just wrong, because some of those customers would have paid anyway. So Recoup holds back a randomized control group and reports the difference instead. Watch what that actually does to the number: on this batch, it doesn't clear statistical significance. So Recoup says that. On camera. Instead of a bigger number, you get a true one — even when the true one is 'we can't prove this yet.'"*

**On screen:** the two numbers side by side, the raw one large and the incremental one small and honestly bounded; hover shows the z-test, CI, MDE, n_t/n_c, the "NOT SIGNIFICANT" marker in the same red typography a real failed test gets — nothing softened for the camera.

> *"That refusal is the whole pitch. A tool that always finds a win isn't measuring anything — it's decorating. Every number on this dashboard, good or null, comes from the same undecorated pipeline."*

---

### 1:00–1:45 · Diagnosis and restraint

> *"Before it acts, Recoup asks two questions almost no recovery bot asks. First: why did this actually fail? A revoked mandate and an OTP timeout have nothing in common except the word 'failed'."*

**On screen:** case detail — root cause with calibrated confidence and SHAP bars.

> *"Second — and this is the one that matters — will contacting this person change anything? This customer would have paid anyway. We leave them alone, and we don't take credit for them. This one has an eight percent uplift on ninety rupees, and chasing costs four. Negative expected value. We don't chase, and here's the arithmetic."*

**On screen:** a case marked `sure_thing`, then one terminating `abandoned_uneconomic` with its EV ledger visible.

---

### 1:45–2:30 · Bounded execution

> *"Everything it does is bounded. Quiet hours: blocked, rule REG-COMM-01. Customer opted out: blocked, permanently, across every case. Revoked mandate: it will never retry that — it sends a re-authorization link instead."*

**On screen:** compliance view — blocked actions each showing the rule ID that fired.

> *"Above the merchant's threshold, a human approves. And even approved actions sit in a cancellable window — watch."*

**On screen:** approve a card, then cancel the staged action before it sends. Then hit the kill switch; everything in flight halts, logged with actor and timestamp.

---

### 2:30–3:05 · The customer side

> *"From the customer's side it has to be one tap. Here's the message on my phone."*

**On screen:** real phone. Open the simulated SMS, tap the single-use link, land on a page that already knows what failed and offers the right fix, complete a Razorpay **test-mode** payment.

> *"And the case flips to recovered, live, on the dashboard."*

**On screen:** cut back to the dashboard updating over SSE within a second or two.

---

### 3:05–3:35 · Hinglish voice and promise-to-pay

> *"For high-value cases, it calls — in Hinglish, inside a bounded conversation graph, not an open-ended chat."*

**On screen / audio:** play 15 seconds of the call. Show the transcript with the node path highlighted.

> *"The customer said they'd pay Friday. Recoup captured that as a structured commitment — and now watch what it does. Nothing. It goes quiet until Friday, then checks whether the promise was kept, and remembers either way."*

**On screen:** case timeline showing `awaiting_promise` and the suppressed escalation.

---

### 3:35–4:05 · Grounded audit

> *"Everything is an event in a hash-chained log. So you can ask it questions."*

**On screen:** type *"Why did we contact this account three times?"* → answer with clickable citations to specific event IDs.

> *"And when the answer isn't in the log —"*

**On screen:** type *"Was this customer happy about the call?"* → explicit refusal naming what would need to be logged.

> *"It refuses. That's a test in CI: twenty questions it can answer, twenty it can't. A fabricated citation fails the build."*

---

### 4:05–4:35 · Break it

> *"The brief asks how you handle failure. So there's a button that breaks it."*

**On screen:** the chaos control. Replay a duplicate webhook — suppressed, zero duplicate contact. Kill a worker mid-action — resumes, no duplicate effect. Take the provider down — circuit breaker opens, case rerouted, not lost. Make the model time out — degraded mode, deterministic template, recovery continues.

> *"Zero duplicate charges, zero duplicate messages, zero lost cases. Every failure ends up in the exception queue where a human can see it."*

---

### 4:35–5:00 · The number and the ask

> *"Two thousand cases, seed forty-two, reproducible with one command on a clean machine. On this run, the honest answer is: not statistically significant — the confidence interval still crosses zero. We're showing you that, not a nicer number from a different run. The method is what we're asking you to trust, and the method just proved itself by refusing to lie to you thirty seconds ago."*

(Verified against a real `make demo` run — see the README's headline block. Re-check this line against your own machine's output before recording: `make demo` prints the exact figures live, and this script should never quote a number that hasn't actually been produced by the code. If your own run comes back significant, say the real number instead — the point is fidelity to whatever the pipeline actually printed, not this specific outcome.)

**On screen:** the headline block, then the repo, then this line, held for three seconds:

> *"It runs on synthetic data, so these numbers describe our data-generating process, not production. The method is what transfers. Everything — the FRD, the compliance matrix, the evaluation protocol and the incident log of what broke while building it — is in the repo."*

---

## Production notes

- **Record the demo before the voice-over.** Get one clean run, then narrate over it. Do not try to talk and drive simultaneously.
- **Pre-warm everything**: models loaded, LLM cache warm for the demo cases, browser tabs open, phone on screen mirror, DND on.
- **Record a backup take on Day 7** so Day 8 is not a first attempt.
- **Cut ruthlessly to 4:50.** Under-running reads as confidence; running over reads as sprawl.
- Full-frame 1080p, 30fps, system audio for the call playback, mic close and level-checked.
- Caption the key numbers on screen — many judges scrub with the sound off.

## The three sentences a judge should be able to repeat afterwards

1. *"They held out a control group, so their recovery number is real."*
2. *"It refuses to act — on uneconomic cases, on opted-out customers, on revoked mandates — and it proves it with tests."*
3. *"They broke it on purpose, on camera, and it held."*
