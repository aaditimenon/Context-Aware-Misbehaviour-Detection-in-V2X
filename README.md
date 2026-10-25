# Context-Aware misbehaviour Detection in V2X 

A hybrid misbehaviour detector for vehicular networks (V2X). It combines **physics-based plausibility checks**, **per-pseudonym window features**, **multi-radius neighbourhood context**, an **environment (road) prior** and **XGBoost** classifiers, with a **distance-weighted cooperative vote** among neighbours.
---

## 1. Problem

V2X messages (BSMs) are authenticated, but authentication does not guarantee the content is *correct*. A certified vehicle can still send false positions or speeds, flood the channel, replay old data, or pretend to be several vehicles (Sybil). **misbehaviour detection** finds such vehicles from message content and context.

## 2. Dataset

[VeReMi Extension](https://hal.archives-ouvertes.fr/hal-02492739) (Kamel et al., 2020), used through a Kaggle re-export:

| Item | Value |
|---|---|
| Messages | 3,194,808 |
| Real vehicles (`sender`) | 24,663 |
| Pseudonyms (`senderPseudo`) | 118,909 |
| Classes | 20 (Normal + 19 misbehaviour types) |
| Normal share | about 59.5% |

**Important data decisions**
- **Tracking uses `senderPseudo`** (what a receiver really sees). The real `sender` ID is used **only** for labels and for splitting train/test by vehicle. Tracking by the real ID would let a detector "see through" Sybil identities, which is unrealistic.
- **Splits are by vehicle**, never by message, so one vehicle never appears in both train and test.
- **Unseen attacks:** Sybil classes (14 to 16) and classes 17 to 19 are never used for training.

### A data-quality finding (why the dataset changed)
I first tried a different Kaggle CSV (22M rows, no sender IDs). Every model, including XGBoost on 2M rows with physics and neighbourhood features, scored **AUC 0.50 to 0.57**, and the labels showed signs of being detached from message content (for example, an identical number of labels swapped between two time bands in every scenario). I concluded that dataset was unusable and moved to the VeReMi Extension export, where the same "can the labels be learned at all?" test gave AUC 0.81 (raw columns) and 0.98 (adding per-sender physics) on unseen vehicles. *(That 0.98 used real sender IDs and is only a learnability check, not a result of the final detector.)*

## 3. Method

```
raw message -> [1] physics checks -> [2] window features -> [3] neighbourhood context
            -> [4] road prior     -> XGBoost (binary / multi-class) -> [5] cooperative vote
            -> vehicle verdict (majority vote over a pseudonym's messages)
```

1. **Physics checks (scores, not flags):** distance moved vs. claimed speed, speed change vs. claimed acceleration, acceleration limit (9 m/s²), heading unit-vector check, speed/heading alignment, noise-field sizes, message gap, per-second message count.
2. **Window features:** mean and max of the main suspicion scores over the last N messages of one pseudonym (N = 5, 10, 20).
3. **Neighbourhood context:** for each message, other pseudonyms heard in the same second, after dead-reckoning positions to a common time. Multi-radius (10, 50, 150, 300 m) counts, speed deviation and heading agreement, plus a **noise-aware Sybil overlap check** (nearest-neighbour distance divided by combined GPS noise; count of others within 5 m, shorter than a car).
4. **Road prior (environment context):** number of *normal training* vehicles seen in each 10 m map cell (the vehicle's own contribution is subtracted, and only training vehicles are used, so nothing leaks).
5. **Cooperative vote:** a message's score is mixed with the distance-weighted score of neighbours within 150 m (weight 1/(d+5)); the mixing weight is chosen on validation.
6. **Vehicle verdict:** a pseudonym is flagged if more than half of its messages are flagged (majority vote). **Latency** = number of messages sent before the first flag.

Baselines: physics-only model, Isolation Forest (unsupervised, fitted on normal training messages only).

## 4. Results

### 4.1 Multi-class classification (known attack types, unseen vehicles)
Test set: 383,110 messages from vehicles never seen in training.

| Metric | Value |
|---|---|
| Accuracy | 0.975 |
| **Macro F1** | **0.922** |
| Normal recall | 0.998 |

Per-class F1: ConstPos 0.995, RandomPos 0.992, RandomPosOffset 0.990, RandomSpeed 0.994, RandomSpeedOffset 0.981, DoSDisruptive 1.000, ConstPosOffset 0.884, Disruptive 0.902, ConstSpeed 0.928, ConstSpeedOffset 0.923, DoS 0.826, DataReplay 0.797, DoSRandom 0.709.

*(Accuracy alone is misleading here because about 74% of test messages are Normal. Macro F1 weights all classes equally.)*

### 4.2 Ablation: environment prior
| | ConstPosOffset recall | ConstPosOffset F1 | Macro F1 |
|---|---|---|---|
| Without road prior | 0.087 | 0.157 | 0.873 |
| **With road prior** | **0.799** | **0.884** | **0.922** |

A constant position offset keeps the track internally consistent (speed matches movement), so single-vehicle physics cannot see it. The road prior exposes it because the claimed position lands where normal cars do not drive.

### 4.3 Binary detection on unseen attacks (Sybil) and latency
<!-- FILL IN from notebook Section 10 before using these claims anywhere. -->

| Model | Test set | Level | Precision | Recall | F1 | FPR | AUC |
|---|---|---|---|---|---|---|---|
| A: physics | unseen Sybil | vehicle | | | | | |
| B: + window | unseen Sybil | vehicle | | | | | |
| C: + neighbourhood | unseen Sybil | vehicle | | | | | |
| D: + cooperative vote | unseen Sybil | vehicle | | | | | |
| E: Isolation Forest | unseen Sybil | vehicle | | | | | |

| Window size N | F1 (unseen Sybil) | Detected attackers | Median latency (messages) |
|---|---|---|---|
| 5 | | | |
| 10 | | | |
| 20 | | | |

## 5. Limitations

- **Labels are per vehicle**: an attacker is malicious from its first message, so "latency" means messages until the first flag, not time since an attack began.
- **No receiver IDs** in this export: the neighbourhood is everything heard in the same one-second window, not a single receiver's view, and the cooperative vote is among neighbouring pseudonyms rather than reports from several receivers.
- **Simulation runs may overlap in time** inside the merged file, so some "neighbours" could belong to a different simulation.
- **Weak classes:** DoSRandom (recall 0.58), Disruptive (0.82), DoS and DataReplay (confused with each other, though both are still flagged as attacks), and about 20% of ConstPosOffset messages are still labeled Normal.
- **Classes 17 to 19 are undocumented** on the Kaggle page, so they are reported only as "unknown attacks".
- **Third-party re-export** of the dataset with an unspecified license; results should be reproduced on the original release.
- Single train/validation/test split by vehicle (no repeated runs or confidence intervals yet).

## 6. Future work

- Read the attack definitions in the VeReMi Extension paper and design targeted features for Disruptive, DoSRandom, DoS and DataReplay.
- Reproduce on the original per-vehicle logs with receiver IDs to enable **true multi-receiver, trust-weighted voting** and a receiver-side view.
- Sequence models (GRU/LSTM/Transformer) over per-pseudonym tracks and graph models over neighbourhoods.
- Repeated runs with confidence intervals, and leave-one-attack-type-out evaluation.
- Pseudonym-change handling and map-matched road context.

## 7. How to run

1. Create a Kaggle notebook with a GPU and add the dataset *VeReMi_Extension_Clean_Data* (`mixalldata_clean.csv`).
2. Run `notebook0_dataset_check.ipynb` to verify the data and labels.
3. Run `notebook1_misbehaviour_detection.ipynb` top to bottom (window-size comparison in Section 12 is the slowest part).

```
.
├── notebook0_dataset_check.ipynb      # data and label sanity checks
├── notebook1_misbehaviour_detection.ipynb
├── results_table.csv                  # written by the notebook
└── README.md
```
