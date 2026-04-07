---
layout: post
title: Fraud detection system
categories: reading
---

Rule-based fraud detection is slow to adapt, plagued by false positives, fragmented across platforms, and blind to novel attack patterns. This post outlines an ML-driven replacement: a five-layer pipeline that ingests cross-platform telemetry, engineers behavioral and temporal features through a hybrid Lambda architecture, runs a four-model ensemble (LightGBM, LSTM, Isolation Forest, Meta-Learner), and serves decisions under 100ms p95 with automatic fallback. It also covers class imbalance handling, drift detection via PSI and proxy signals, and a canary-based retraining pipeline designed to keep the system current as fraud techniques evolve.

## Problem Framing: From Rule-Based to Intelligent Detection

The rule-based fraud detection system struggles with four core challenges:

- **Slow rule creation** — Writing new rules takes weeks, leaving the system unable to keep up with emerging fraud techniques.
- **High false positive rate** — Legitimate users get flagged too often, but lowering the detection threshold means more fraud slips through.
- **Platform fragmentation** — Separate rule sets for Browser, iOS, Android, and Desktop create heavy maintenance overhead and inconsistent coverage.
- **Class imbalance** — Fraudulent transactions make up a very small share of traffic (often < 2%), so traditional models tend to favor the majority class and miss subtle anomalies.

Rule-based detection is inherently reactive: the team must observe a new fraud pattern before they can write a rule to block it, meaning the system is always one step behind attackers. As rules accumulate and conflict with each other, maintenance costs grow — and the system remains completely blind to novel fraud behavior until someone intervenes manually.

This is why ML is now the right move. The goal isn't simply to swap rules for a model, but to build a fundamentally proactive system — one that learns directly from data to detect new spoofing techniques faster, reduces false positives without sacrificing recall, unifies detection logic across all platforms, and adapts continuously with minimal operational overhead.

## System Architecture Overview

<img src="/img/fraud-detection/image.png"/>

The system is structured as a five-layer pipeline, designed to ingest raw telemetry from multiple platforms, transform it into model-ready features, run ensemble inference, and serve predictions in production with continuous monitoring.

**Layer 1 — Multi-Platform Data Sources**:
Telemetry is collected from four surfaces: Browser, iOS, Android, and Desktop. Unifying signals across these platforms is foundational to the cross-platform detection goal and eliminates the fragmentation inherent in the previous rule-based approach.

**Layer 2 — Data Pipeline Layer:**
Incoming data is processed through two parallel pipelines. The batch pipeline handles ETL processing and writes to a Data Lake for historical storage and model training. The real-time pipeline uses Kafka Streaming to compute features on the fly, enabling low-latency inference at serving time.

**Layer 3 — Feature Store Layer:**
The feature store acts as the central hub between data and models, with three responsibilities: storing historical features with versioning support, serving features in real time with caching for low-latency access, and engineering cross-platform feature mappings with automated selection to ensure consistency across all input surfaces.

**Layer 4 — ML Model Ensemble:**
Four models operate in concert. LightGBM handles tabular, structured signals. LSTM captures sequential behavioral patterns over time. Isolation Forest provides unsupervised anomaly detection, particularly valuable given the severe class imbalance (~2% fraud rate). A Meta-Learner acts as the final combiner, aggregating the outputs of the three base models into a single decision score.

**Layer 5 — Online Serving & Monitoring:**

Predictions are served with a strict sub-100ms latency target and auto-scaling to handle traffic spikes. A monitoring module tracks both model performance metrics and business KPIs in tandem. A dedicated drift detection component watches for both model drift and data drift, triggering retraining or alerting when the input distribution or model behavior shifts significantly from baseline.

## Data Pipeline & Feature Store

![alt text](/img/fraud-detection/image-1.png)

All telemetry events from the four platform SDKs are first funneled into Kafka, which acts as the central message broker and decouples data ingestion from downstream processing. From Kafka, data flows into two parallel pipelines that serve distinct purposes.

**The Batch Pipeline**, orchestrated by Airflow, runs daily ETL jobs to process and consolidate raw events into structured training data. This offline path is optimized for completeness and correctness over latency, feeding historical features into BigQuery with point-in-time correctness — ensuring that no future information leaks into training labels, a critical requirement for fraud model integrity.

**The Real-time Pipeline**, powered by Kafka Streaming or Flink, computes features continuously over rolling windows of 5, 30, 180 minutes or 1 day. These multi-resolution windows allow the model to capture both immediate behavioral signals — such as a sudden burst of login attempts — and slower-developing patterns that only become detectable over a longer horizon.

**Hybrid Lambda Architecture:** Long-term windows (such as 1 day, 1 week, or longer) are pre-computed from the Offline Store and indexed back into the Online Store. At the time of serving, the system aggregates these indexed historical values with the most recent real-time events from the streaming pipeline.


Both pipelines converge at the Feature Store, which is split into two stores with complementary roles. The offline store (BigQuery) maintains the full historical record used for model training and experimentation. The online store (NoSQL cache) holds the most recent feature values and indexed historical data, which is optimized for serving, targeting a p95 latency of ~10ms to stay well within the end-to-end 100ms budget.

The health of this layer is tracked through three performance metrics: stream latency, feature freshness, and data completeness. While the batch pipeline prioritizes completeness and point-in-time correctness to ensure high-quality training labels, the stream pipeline focuses on sub-second latency for immediate signals. Monitoring completeness % is especially critical, as missing or delayed features at serving time can cause 'silent failures'—degrading model predictions without triggering explicit system errors.


## Signals: Prioritized Browser Telemetry
Feature selection follows a deliberate prioritization framework, grouping signals into three tiers based on their detection value, compute cost, and implementation complexity.

**P1 signals** form the core detection baseline and are the first to be implemented. WebGL/Canvas fingerprinting exposes GPU emulation and headless browser environments commonly used by automated attackers. Mouse movement dynamics exploit the fact that bots generate mechanically inhuman interaction patterns that are difficult to convincingly fake at scale. IP versus timezone/locale mismatch is a lightweight but high-signal indicator of VPN and proxy usage — notable for its low compute cost relative to its discriminative power. Beyond these, several additional P1 signals provide strong coverage at minimal overhead: User-Agent consistency checks detect mismatches between the reported browser string and actual runtime capabilities; screen resolution versus reported display values frequently reveal emulated environments; cookie and localStorage availability exposes abnormal storage behavior common in headless setups; JavaScript execution timing flags bots and emulators that process scripts at unnatural speeds; referrer chain anomalies indicate direct scripted access with no legitimate navigation history; and unusually low device memory or CPU core counts — such as a single core with 256MB RAM — are reliable indicators of an emulated environment rather than a real consumer device.

**P2 signals** add a behavioral biometrics and environmental layer once the P1 baseline is validated. Keystroke dynamics differentiate genuine human typing from scripted form-filling. Browser API anomalies target specific automation frameworks — particularly Selenium and Puppeteer — which leave detectable traces in how they interact with browser APIs. Network characteristics based on ASN type provide an additional, complementary channel for proxy and VPN detection beyond IP geolocation alone. Several further P2 signals strengthen this layer: mixing of touch and mouse events in ways that are physically inconsistent reveals automation on mobile-emulated sessions; scroll behavior patterns distinguish humans — who scroll irregularly and with variable velocity — from bots that move at constant, mechanical speed; form interaction timing catches scripted submissions that fill fields far faster than any human could; font enumeration fingerprinting surfaces OS environment inconsistencies that differ from the claimed platform; audio context fingerprinting complements WebGL by detecting emulation at the audio stack level; WebRTC IP leak detection can expose a user's real IP address behind a VPN or proxy via STUN requests; battery API status anomalies — such as a constant or null value — are common artifacts of emulated environments; and unusual clipboard access attempts may signal automation interacting with form fields programmatically.

**P3 signals** address more sophisticated, cross-session attack patterns. Session interaction density captures the overall rhythm of a session to distinguish real users from bots that move through flows at unnatural speeds. Cross-session device consistency detects fingerprint cycling attacks, where fraudsters rotate device identifiers across sessions to evade detection. Additional P3 signals extend coverage to account-level and temporal patterns: velocity features such as login attempts per hour leave clear footprints in credential stuffing campaigns; a mismatch between account age and observed behavior — for instance, a newly created account exhibiting power-user navigation patterns — is a strong anomaly signal; device-to-account binding consistency flags cases where a single device accesses many distinct accounts, a common signature of account takeover operations; geographic jump detection identifies impossible travel scenarios where the same account appears from two geographically distant IPs within an implausibly short time window; canvas noise consistency tracked over time reveals fingerprint rotation, since legitimate users show stable rendering while attackers cycle values; an unusual absence of any browser extensions may indicate a clean headless environment rather than a real user's browser; and deviations from a user's established time-of-day activity baseline provide a behavioral anchor that is difficult for attackers to replicate consistently.

Three key design decisions govern how these signals are used in practice. First, the rollout follows a start simple principle: P1 signals alone are expected to deliver a strong performance baseline, and P2/P3 complexity is only introduced after that baseline is empirically validated — avoiding premature over-engineering. Second, privacy considerations apply particularly to mouse and keystroke biometrics, audio fingerprinting, and font enumeration, all of which are classified as sensitive behavioral data; collection must be governed by a proper consent framework and subject to data minimization requirements that vary by jurisdiction, and legal review is recommended before enabling collection in regulated markets. Third, a missing data strategy is necessary because browser restrictions on iOS Safari and Firefox may block access to certain APIs — including Battery API and WebRTC — and the system must handle these gaps gracefully through imputation rather than failing or producing unreliable scores. Finally, as the feature set expands into P2 and P3 territory, correlation risk becomes a practical concern: signals such as WebRTC leak and ASN type carry overlapping information, and systematic feature selection is needed to avoid redundancy that inflates model complexity without improving predictive power.

## Feature Engineering
Features are organized into four distinct layers, each capturing a different dimension of user and device behavior, and designed to be introduced incrementally as the system matures.

The Static layer forms the foundation, encompassing device fingerprint, browser capabilities, IP/ASN, and geolocation. These features are inexpensive to compute and highly stable over time — they change rarely for legitimate users but show clear anomalies when attackers rotate environments or spoof their identity.

The Aggregated layer moves from point-in-time snapshots to frequency-based signals, including sessions per user/device, failed attempt counts, and requests per IP per hour. This layer is purpose-built for frequency anomaly detection, making it particularly effective at catching credential stuffing and brute-force patterns that only become visible when activity is aggregated over a window rather than evaluated in isolation.

The Temporal layer introduces time-awareness through features such as time between actions, rolling windows at 5, 30, and 180-minute resolutions, and time-of-day baselines. Together these enable sequential pattern detection — capturing not just what a user does, but the rhythm and ordering of those actions, which is far harder for automated systems to replicate convincingly.

The Behavioral layer is the most expressive and the most complex, relying on LSTM-based sequence modeling and mouse/keystroke dynamics to construct a latent representation of each user's behavioral fingerprint. Because of its cost and data requirements, this layer is scoped to Phase 2, to be introduced only after the first three layers have been validated in production.

Key design decision: Layers 1–3 combined with P1–P2 signals constitute the MVP. This staged approach ensures the team can measure incremental gains at each step and avoid the trap of building a complex system that is difficult to debug when something goes wrong.

Beyond individual features, three cross-feature interactions deserve special attention as they encode some of the most discriminative fraud signals in the entire feature set. A mismatch between device timezone and IP country is a strong, low-cost indicator of VPN or proxy use. The presence of mouse events on a device that only supports touch is a near-certain automation indicator, since no legitimate mobile user would generate such events. Finally, off-hours activity that deviates sharply from a user's established personal pattern — particularly when combined with other anomalies — is a reliable account takeover signal, as attackers operating across time zones rarely match the behavioral baseline of the account's true owner.

## Model Architecture
The inference pipeline follows a linear flow: raw features pass through pre-processing, enter the ensemble layer, and produce a final fraud score accompanied by a SHAP explanation for each decision.
Rather than relying on a single model, the system is designed as a four-component ensemble, where each model targets a distinct class of fraud pattern and compensates for the blind spots of the others.

The Tree model (LightGBM / XGBoost) serves as the workhorse of the ensemble. It handles tabular features well, trains efficiently, and critically, produces decisions that are directly interpretable via SHAP — making it the primary tool for analyst review and regulatory compliance. Its fraud targets are device fingerprinting anomalies and network-level signals, which map naturally to structured, tabular input.

The DNN (LSTM) handles the sequential dimension that tree models cannot capture. By modeling temporal dependencies across a session, it is well-suited to detecting behavioral bots and the subtle patterns embedded in keystroke and mouse dynamics — signals that only become meaningful when evaluated as an ordered sequence rather than a static snapshot.

Isolation Forest operates without labels, making it the ensemble's defense against novel and zero-day attack patterns. In a domain where fraud techniques evolve continuously and labeled data for new attack types is by definition unavailable at the time of first exposure, an unsupervised anomaly detector provides a critical safety net that purely supervised models cannot offer.

The Meta-Learner — a logistic regression applied to the outputs of the three base models — acts as the final arbiter. Rather than taking a majority vote, it learns to weight and combine the signals from each model, calibrating the final confidence score to reflect the true posterior probability of fraud. This makes the output more reliable as a decision threshold input than any raw model score would be in isolation.

Why ensemble over a single model? Four reasons drive this choice. First, complementary strengths: tree models are fast and effective on known patterns, while Isolation Forest catches unknowns that carry no labels yet. Second, adversarial robustness: a fraudster who successfully evades one model is unlikely to evade all three simultaneously, raising the cost and complexity of any evasion strategy. Third, interpretability: the tree model combined with SHAP provides a human-readable explanation for every flagged session, which is a practical requirement for analyst workflows and compliance obligations. Fourth, phased delivery: the tree model alone constitutes a valid Phase 1 MVP that can go to production quickly, with the LSTM and full ensemble introduced in Phase 2 once the baseline is stable — keeping early delivery risk low while preserving the full architecture as the long-term target.

## Class Imbalance & Decision Threshold
With 98% of sessions being legitimate and only 2% fraudulent, a naïve model can achieve 98% accuracy by predicting "legitimate" for everything — making raw accuracy a completely meaningless metric in this context. Addressing class imbalance is therefore not a preprocessing detail but a core architectural concern that touches every layer of the system.

The solution is a three-layer strategy that intervenes at the data level, the algorithm level, and the decision level simultaneously.

At the data level, BorderlineSMOTE combined with Tomek Links is used to rebalance the training distribution. BorderlineSMOTE synthesizes new minority-class examples specifically near the decision boundary — the hardest and most informative region — rather than oversampling uniformly across the fraud class. Tomek Links then clean the majority class by removing borderline legitimate samples that sit too close to fraud examples, sharpening the boundary. Critically, full 1:1 resampling is avoided: over-resampling to perfect balance causes the model to overfit on synthetic samples, and the optimal resampling ratio is treated as a hyperparameter tuned empirically via cross-validation on PR-AUC rather than set by convention.

At the algorithm level, scale_pos_weight in LightGBM is set to reflect the class imbalance, and Focal Loss is applied for the LSTM component. Both techniques direct the model's attention toward hard and borderline cases during training — the exact examples where the decision boundary matters most — rather than allowing easy legitimate-class examples to dominate the gradient updates.

At the decision level, the classification threshold is tuned on a held-out validation set to minimize business cost rather than to maximize any single model metric. This is where the cost-sensitive decision framework becomes the governing logic. A False Negative — missed fraud — carries a high cost because it translates directly to financial loss. A False Positive — a wrongly blocked legitimate user — carries a lower but non-negligible cost in the form of manual review overhead and user friction. A True Positive carries a small investigation cost but is net positive. A True Negative carries zero cost. Given this asymmetry, the threshold is deliberately set below 0.5 to bias toward catching fraud at the expense of accepting more false positives, with the false positive volume kept within the capacity of the manual review queue.

The key metric governing all modeling decisions is Precision@90%Recall — fixing recall at 90% as a non-negotiable floor that ensures the vast majority of fraud is caught, then maximizing precision within that constraint to control the false positive rate. PR-AUC is used over ROC-AUC throughout evaluation because ROC-AUC is well known to produce overly optimistic results on imbalanced datasets, where the large number of true negatives inflates the curve regardless of how poorly the model performs on the minority class.

## Online Serving — Architecture & Reliability
Every client request passes through a carefully sequenced pipeline designed to deliver a fraud decision within 100ms at p95, while remaining resilient to model failures without ever leaving the calling system without a response.

The serving pipeline flows as follows. 

![alt text](/img/fraud-detection/image-2.png)

Incoming requests first hit a Load Balancer that handles A/B traffic splitting for safe model rollouts. They then pass through an API Gateway before reaching the Input Validation step, which rejects malformed or incomplete payloads early — before any expensive computation is triggered. Feature retrieval follows, pulling pre-computed values from Redis cache backed by the Feature Store; a cache hit keeps this step to approximately 10ms, while a cache miss falls back to the Feature Store at around 30ms. The ensemble inference step then runs the four-model stack and produces a raw fraud score, which is subsequently passed through a Business Rule layer that applies any hard overrides — such as blocklists or regulatory constraints — before the final decision is made. The last component in the chain is the Circuit Breaker, which monitors ML system health in real time: if inference is healthy, the decision flows out normally; if the ML layer fails, the circuit breaker automatically falls back to the rule-based system, ensuring the service never goes dark.

The latency budget is structured around two paths. A cache hit path completes in approximately 70ms end-to-end, comfortably within the 100ms p95 target. A cache miss path — requiring a live Feature Store lookup — extends to approximately 90ms, still within budget but leaving little margin, which underscores why cache hit rate is an operationally critical metric.

## Drift Detection, Alerting & Business Metrics

The core challenge of drift monitoring in a fraud detection context is that labels arrive days to weeks after the session occurs — a fraudulent transaction may not be confirmed until a chargeback is filed or an investigation is completed. This means the system cannot wait for ground truth to detect degradation; it must rely on proxy signals in real time to catch problems before they compound into significant business impact.

Detection is structured across four layers, each monitoring a different aspect of system health.
Input drift tracks the distribution of incoming features using Population Stability Index (PSI), with a threshold of PSI > 0.25 triggering an alert. A sudden spike in null rates is monitored in parallel, as missing feature values at serving time are often the first observable sign that a data pipeline has broken or that an attacker is deliberately probing the system with malformed inputs.

Prediction drift monitors the distribution of the model's output score bins rather than the features themselves. A PSI > 0.25 on score distributions indicates that the model is behaving differently — even if individual features appear stable — which can happen when attackers subtly shift their behavior in ways that individually fall below feature-level thresholds but collectively move the score distribution.

Proxy performance provides a near-real-time approximation of model quality before confirmed labels are available. Two signals serve this purpose: disagreement between the rule-based system and the ML model, and the rate at which sessions are escalated for manual review. When either exceeds 2× its baseline, it is a strong indicator that something has changed — whether in the fraud landscape, the model, or the data pipeline.
Confirmed performance is evaluated weekly and monthly once labels have accumulated, using Precision@90%Recall, PR-AUC, and F1 as the governing metrics. These provide the ground-truth view of model quality that proxy signals can only approximate.

The alerting framework translates each detection signal into a concrete operational response:

| **Alert / Metric** | **Condition** | **Action** |
|---|---|---|
| **Model degradation** | Model metrics drop below agreed target | Trigger retraining |
| **PSI feature drift** | Top feature PSI > 0.25 | Investigate + expedite retrain |
| **High FP rate** | Exceeds agreed threshold | Review decision threshold |
| **Null rate spike** | Behavioral feature nulls spike | Possible adversarial input — investigate |
| **Fraud block rate ↓** | Sudden drop in block rate | Evasion signal — investigate immediately |
| **Rule-only vs ML catch rate** | Compared weekly/monthly | Validates that ML adds measurable value over rules |
| **Rule-only vs ML catch rate** | Compared weekly/monthly | Validates that ML adds measurable value over rules |

Two alerts deserve particular attention. A sudden drop in fraud block rate is not a positive signal — it is most likely an evasion signal, meaning attackers have found a way to score below the detection threshold and the drop reflects successful evasion rather than reduced fraud activity. This warrants immediate investigation rather than passive monitoring. Similarly, a null rate spike on behavioral features is a dual-concern signal: it could indicate a data pipeline failure, but it could equally indicate adversarial probing where attackers deliberately omit signals they believe the model relies on — making it a potential indicator of an active, informed attack on the detection system itself.

## Retraining Pipeline & Adversarial Strategy
Keeping a fraud detection model current is not a one-time effort — it is an ongoing operational discipline. As fraud patterns evolve and the input distribution shifts, a model trained on historical data will degrade silently unless a structured retraining pipeline is in place to refresh it continuously.

The end-to-end retraining pipeline follows five stages. It begins with Data Preparation, pulling a recent time window of labeled sessions to ensure the training distribution reflects current fraud behavior rather than patterns that may have already been abandoned by attackers. The Training stage retrains the model on this refreshed dataset, preserving the same architecture and hyperparameter space unless a deliberate upgrade is warranted. Validation goes beyond standard technical metrics — the retrained model is evaluated against both held-out performance metrics and a dedicated adversarial test set designed to probe known evasion techniques, ensuring the new model does not regress on attack patterns the system has previously learned to catch. Deployment follows a progressive canary rollout: 5% of live traffic first, expanding to 20%, then 50%, and finally 100% — giving the team sufficient signal at each stage to detect regressions before they affect the full user base. The final stage is continuous Monitoring, where the new challenger model is compared against the current champion daily, with auto-promote and auto-rollback logic that removes the need for manual intervention on routine updates while preserving human oversight for edge cases.

Retraining is triggered through four distinct mechanisms, balancing routine hygiene with reactive response:

| **Trigger** | **Type** | **Threshold** |
|---|---|---|
| **Scheduled** | Routine | Periodic cadence to keep the model fresh on recent fraud patterns |
| **PSI drift alert** | Reactive | Any top-10 feature with PSI > 0.25 |
| **Rule-ML disagreement spike** | Reactive | > 2× baseline disagreement rate between rule engine and ML model |
| **Manual trigger** | Reactive | DS team judgment — e.g. awareness of a known new fraud campaign |