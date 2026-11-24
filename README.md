# Pose-ReID

**Tracking-assisted person re-identification using YOLO26x-Pose, body-ratio analysis, OpenCV and database-assisted identity matching.**

> **Note:** This repository contains documentation only. The source code and implementation details are not published because the project was developed in a professional environment and is subject to confidentiality restrictions. Everything below describes the system's objectives, architecture and workflow at a level the project owner is permitted to disclose.

---

## What it does

Pose-ReID detects people in a video stream, tracks them across frames, and re-identifies them when the tracker loses them and later assigns a new tracking ID.

Instead of storing raw pixel measurements or image embeddings, the system represents each person by **shoulder-normalized body proportions** derived from human body keypoints, backed by **shirt colour** as a secondary appearance cue.

The design cleanly separates two kinds of identity:

| Identifier | Owner | Lifetime |
|---|---|---|
| **YOLO tracking ID** | The tracker | Temporary — changes when a person is lost and re-detected |
| **Person ID** | The system database | Persistent — the person's real identity |

When a person reappears with a new tracking ID, the system matches the new detection against stored person records. On a match, the new tracking ID is associated with the existing Person ID and that association is saved, so subsequent frames need no further database search.

---

## Architecture

```
Camera frames
     │
     ▼
YOLO26x-Pose ──► person detections + body keypoints
     │
     ▼
YOLO Tracking ──► temporary tracking IDs
     │
     ▼
OpenCV ──► body measurements ──► shoulder-normalized body ratios
     │
     ▼
Database ──► body-ratio records, person records,
             tracking-ID → person-ID associations
     │
     ▼
OpenCV ──► custom bounding boxes + person-identity tags
```

### Body-ratio representation

Shoulder width is the reference measurement and is assigned a ratio of `1.0`. Other selected measurements are expressed relative to it:

```
normalized body ratio = body measurement / shoulder width
```

Working with ratios rather than raw pixel distances reduces the effect of a person's apparent size in the image. Ratios are still affected by pose, camera perspective, viewing angle, keypoint quality and occlusion — which is why the system keeps multiple samples per person.

### Multi-sample body records

Each person can hold up to **30 body-ratio records**, so different observed body configurations are represented instead of a single average.

Update logic:

1. Compute the current normalized body ratios from keypoints.
2. Compare them with the person's existing records.
3. If the new data differs by more than **10%** from *every* existing record, store it as a new record.
4. Stop at 30 records.

> The stored records are body ratios calculated from keypoints — not raw poses or raw keypoint coordinates.

### Re-identification flow

**While tracked:** the tracking-ID → person-ID association is reused directly; no per-frame database comparison.

**When lost and re-detected with a new tracking ID:**

```
Before loss:   Tracking ID 1  →  Person A
Lost by tracker.
Reappears:     Tracking ID 8
Matching:      new body ratios + shirt colour vs. stored records
Result:        Tracking ID 8  →  Person A   (association saved)
After that:    Tracking ID 8 identified from the saved association
```

*(Illustrative IDs, not project records.)*

### Why shirt colour

Body proportions alone can mismatch different people with similar builds, or when keypoints are unreliable at distance. Shirt colour adds an appearance-based cue to disambiguate those cases.

### Visualization

Raw YOLO detection boxes are used internally only. OpenCV draws the system's own bounding boxes and tags, and the tag shows the **system person identity**, not the raw tracking ID.

---

## Tech stack

| Technology | Role |
|---|---|
| **Python** | Main implementation language |
| **YOLO26x-Pose** | Person detection and body-keypoint prediction |
| **YOLO tracking** | Frame-to-frame tracking and tracking IDs |
| **OpenCV** | Keypoint processing, measurement calculation, custom visualization |
| **Database** | Person records, body-ratio records, tracking-ID ↔ person-ID associations |

---

## Engineering concepts demonstrated

- Human pose estimation and keypoint processing
- Skeletal body-measurement calculation
- Shoulder-width normalization for scale invariance
- Per-person reference-data management with multiple samples
- Threshold-based record insertion
- Multi-object tracking across video frames
- Tracking-ID ↔ persistent-ID association
- Tracking-assisted re-identification (track first, match only when needed)
- Appearance-based matching as a secondary cue
- Database-assisted identity management
- Custom computer-vision visualization

---

## Scope and limitations

- Body ratios are pose- and viewpoint-sensitive; occlusion and low-resolution keypoints degrade accuracy.
- No published benchmark results, dataset or evaluation metrics accompany this project.
- Matching is not intended as biometric identification — it is a tracking aid for continuity within the observed video context.

---

## Status

Documentation-only repository. Implementation, datasets, internal screenshots and configuration are withheld for confidentiality reasons.

---

## License

No source code is distributed here, so no open-source license is granted for the underlying implementation. The text of this README may be reused with attribution.
