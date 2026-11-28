<div align="center">

# Pose-Based Person Re-Identification System

**Tracking-assisted person re-identification across camera feeds using shoulder-normalized body proportions and shirt colour.**

![status](https://img.shields.io/badge/status-documentation--only-lightgrey?style=flat-square)
![language](https://img.shields.io/badge/language-Python-3776AB?style=flat-square&logo=python&logoColor=white)
![model](https://img.shields.io/badge/model-YOLO26x--Pose-8A2BE2?style=flat-square)
![vision](https://img.shields.io/badge/vision-OpenCV-5C3EE8?style=flat-square&logo=opencv&logoColor=white)
![store](https://img.shields.io/badge/store-SQLite-003B57?style=flat-square&logo=sqlite&logoColor=white)

</div>

> [!IMPORTANT]
> **Documentation only.** The source code and implementation details are not published — this project was developed in a professional environment and is subject to confidentiality restrictions. Everything below describes the objectives, architecture and workflow at a level the project owner is permitted to disclose.

## Overview

Pose-ReID detects people in video from multiple cameras, tracks them across frames, and re-identifies them when the tracker loses them — whether they leave one camera's view and appear in another, or step out and back into the same view. Either way, the tracker hands them a new tracking ID, and the system has to work out who they are again.

Multi-camera operation is the production target. Pre-production runs a single camera feed.

Instead of storing raw pixel measurements or image embeddings, the system represents each person by **shoulder-normalized body proportions** derived from human body keypoints, backed by **shirt colour** as a secondary appearance cue.

The design cleanly separates two kinds of identity:

| Identifier | Owner | Lifetime |
|:--|:--|:--|
| **YOLO tracking ID** | The tracker | Temporary — changes when a person is lost and re-detected |
| **Person ID** | The system database | Persistent — the person's identity across cameras and time |

When a person reappears with a new tracking ID, the system matches the new detection against stored person records. On a match, the new tracking ID is associated with the existing Person ID and that association is saved, so subsequent frames need no further database search. With no match, a new Person ID is created.

## How it works

```text
Camera feeds (multiple)
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

Working with ratios rather than raw pixel distances reduces the effect of a person's apparent size in the image — which matters directly here, since the same person is seen at different distances from different cameras. Ratios are still affected by pose, camera perspective, viewing angle, keypoint quality and occlusion, which is why the system keeps multiple samples per person.

### Multi-sample body records

Each person holds up to **30 body-ratio records**, so different observed body configurations are represented instead of a single average.

1. Compute the current normalized body ratios from keypoints.
2. Compare them with the person's existing records.
3. While the person has fewer than 30 records: if the new data differs by more than **10%** from *every* existing record, store it as a new record. The difference is evaluated across the body-ratio record as a whole, not ratio by ratio.
4. At 30 records the set is full and stops growing.

> [!NOTE]
> The stored records are body ratios calculated from keypoints — not raw poses or raw keypoint coordinates.

### Re-identification

**While tracked** — the tracking-ID → person-ID association is reused directly. No per-frame database comparison.

**When an unknown tracking ID appears** — the new detection is matched against stored person records:

```text
New / unknown tracking ID
        │
        ▼
Compare new body ratios + shirt colour
against every stored person record
        │
   ┌────┴─────┐
 match       no match
   │            │
   ▼            ▼
associate    create a new Person ID
tracking     and store this person's
ID → person  body ratios under it
   │
   ▼
save the association; reuse it
for later frames with this ID
```

Worked example *(illustrative IDs, not project records)*:

```
Before loss:   Tracking ID 1  →  Person A
Lost by tracker.
Reappears:     Tracking ID 8   (new ID, no association)
Matching:      body ratios + shirt colour vs. stored records
Result:        Tracking ID 8  →  Person A   (association saved)
After that:    Tracking ID 8 identified from the saved association
```

### Why shirt colour

Body proportions alone can mismatch different people with similar builds, or when keypoints are unreliable at distance — a common situation across camera views. Shirt colour is therefore checked on **every** re-identification, not only as a fallback, and helps disambiguate otherwise close body-ratio matches.

### Visualization

Raw YOLO detection boxes are used internally only. OpenCV draws the system's own bounding boxes and tags, and the tag shows the **system Person ID**, not the raw tracking ID — so a person keeps the same label as their tracking ID changes.

## Tech stack

| Technology | Role |
|:--|:--|
| **Python** | Main implementation language |
| **YOLO26x-Pose** | Person detection and body-keypoint prediction. The `x` variant was chosen for the most accurate keypoints available, since measurement quality depends entirely on them. |
| **YOLO tracking** | Frame-to-frame tracking and tracking IDs |
| **OpenCV** | Keypoint processing, measurement calculation, custom visualization |
| **SQLite** | Person records, body-ratio records and tracking-ID ↔ person-ID associations. Chosen as the pre-production store — simple, file-based, no server to run. |

## Concepts demonstrated

- Human pose estimation and keypoint processing
- Skeletal body-measurement calculation
- Shoulder-width normalization for scale invariance
- Per-person reference-data management with multiple samples
- Threshold-based record insertion
- Multi-object tracking across video frames
- Tracking-ID ↔ persistent-ID association
- Tracking-assisted re-identification — track first, match only when the tracker can't help
- Appearance-based matching as a secondary cue
- Database-assisted identity management
- Custom computer-vision visualization

## Limitations

- **Viewpoint sensitivity.** Body ratios are pose- and viewpoint-sensitive; occlusion, distance and low-resolution keypoints degrade accuracy. A person seen from a very different angle in a second camera is the hardest case — and is not yet validated, since pre-production runs a single camera.
- **No frame skipping.** Every frame is processed.
- **Pre-production store.** SQLite was not chosen for concurrent writes from many cameras or long-term record growth.
- **Tracking ID reuse.** Tracking IDs are assigned by the tracker and can repeat across tracker restarts. Persisted associations assume they stay unambiguous — scoping them per session or per camera is the safe approach.
- **No published metrics.** No benchmark results, dataset or evaluation figures accompany this project.
- **Not biometric ID.** Matching is a tracking aid for continuity across the observed camera feeds, not a biometric identification system.

## Status

Documentation-only repository. Implementation, datasets, internal screenshots and configuration are withheld for confidentiality reasons.

Pre-production runs a single camera. Multi-camera re-identification is the production target and has not yet been validated at that scale.

## License

No source code is distributed here, so no open-source license is granted for the underlying implementation. The text of this README may be reused with attribution.
