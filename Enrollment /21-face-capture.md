# Chapter 21 — Face Capture

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 21.1 What is Face Capture?

**Face capture** is the process of recording a high-quality photograph of a person's face during enrollment that meets international standards for facial recognition accuracy and interoperability.

Face is the **most natural and universally accepted biometric**. Every human recognizes other humans by face — it requires no learned behavior, no physical contact, and no special equipment beyond a camera. In an NBIS, the enrolled face serves three purposes simultaneously:

```
┌──────────────────────────────────────────────────────────────┐
│              Three Roles of Face in NBIS                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. BIOMETRIC TEMPLATE                                       │
│     Face geometry extracted → template stored in CIDR       │
│     Used for: 1:1 verification at eGates, portals          │
│     Used for: 1:N deduplication alongside fingerprints      │
│                                                              │
│  2. VISUAL IDENTITY RECORD                                   │
│     The photograph printed on the ID card / passport        │
│     Used for: visual inspection by officers, border agents  │
│     Used for: document-to-person comparison at checkpoints  │
│                                                              │
│  3. DOCUMENT CROSS-CHECK                                     │
│     Enrolled face compared to face in supporting documents  │
│     (passport photo vs live capture)                        │
│     Used for: fraud prevention during enrollment            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.2 Face vs Other Biometrics

```
┌──────────────────────────────────────────────────────────────┐
│              Face Compared to Fingerprint and Iris           │
├──────────────────────┬───────────────────────────────────────┤
│                      │                                      │
│  ADVANTAGES of Face: │  CHALLENGES of Face:                 │
│                      │                                      │
│  ✅ No contact       │  ⚠️  Sensitive to lighting           │
│  ✅ Works at distance│  ⚠️  Pose variation affects accuracy │
│  ✅ Universally      │  ⚠️  Aging changes appearance        │
│     accepted         │  ⚠️  Glasses, masks, makeup          │
│  ✅ Dual use: ID     │  ⚠️  Documented accuracy disparities │
│     card photo       │      across demographic groups       │
│  ✅ Cheap cameras    │  ⚠️  2D photo spoofing (easier than  │
│  ✅ No training      │      fingerprint spoofing)           │
│     required         │                                      │
│  ✅ Works for        │                                      │
│     children         │                                      │
│                      │                                      │
└──────────────────────┴───────────────────────────────────────┘
```

---

## 21.3 ICAO Standards for Face Capture

The **International Civil Aviation Organization (ICAO)** defines the global standard for facial photographs used in identity documents (passports, national ID cards). ICAO Doc 9303 Part 9 specifies exactly what a compliant face image must look like.

NBIS enrollment must produce ICAO-compliant face images because:
- The same image goes on the ID card (travel / identity use)
- Border systems worldwide expect ICAO format
- ePassport chip stores ICAO-format face image
- ABIS systems are tuned for ICAO-format input

```
┌──────────────────────────────────────────────────────────────┐
│              ICAO Face Image Requirements                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  POSE:                                                       │
│  ├── Frontal: subject faces camera directly                 │
│  ├── Head rotation: ≤ ±5° yaw (left-right)                 │
│  ├── Head tilt: ≤ ±5° roll (ear toward shoulder)           │
│  ├── Head pitch: ≤ ±5° pitch (chin up/down)                │
│  └── Eyes: open, clearly visible, not obscured             │
│                                                              │
│  EXPRESSION:                                                 │
│  ├── Neutral expression (no smile, no frown)                │
│  ├── Mouth closed                                           │
│  └── Relaxed facial muscles                                 │
│                                                              │
│  EYES:                                                       │
│  ├── Both eyes fully open                                   │
│  ├── Eyes looking directly at camera (not sideways)        │
│  ├── No red-eye (use anti-red-eye lighting)                │
│  └── Glasses: generally NOT recommended                     │
│      (glare on lenses + obscures eyes)                     │
│                                                              │
│  BACKGROUND:                                                 │
│  ├── Plain, uniform (white or light gray preferred)        │
│  ├── No patterns, shadows, or objects behind subject       │
│  └── Sufficient contrast with subject's face/hair          │
│                                                              │
│  LIGHTING:                                                   │
│  ├── Uniform — no harsh shadows on face                    │
│  ├── Both sides of face equally lit                        │
│  ├── No overexposure (bright spots wiping out detail)      │
│  └── No underexposure (face too dark)                      │
│                                                              │
│  IMAGE DIMENSIONS:                                           │
│  ├── Minimum: 300×300 pixels                               │
│  ├── Recommended: 600×800 pixels (width×height)            │
│  ├── Face must occupy: 70–80% of frame height              │
│  ├── Eye-to-eye distance: 90 pixels minimum                │
│  └── Color depth: 24-bit RGB (color)                       │
│                                                              │
│  FILE FORMAT:                                                │
│  ├── JPEG 2000 (preferred for ICAO chip storage)           │
│  ├── JPEG (acceptable for printed ID cards)                │
│  ├── PNG (lossless for enrollment archive)                 │
│  └── ISO/IEC 19794-5: Face Image Data standard            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.4 Face Capture Equipment

```
┌──────────────────────────────────────────────────────────────┐
│              Face Capture Equipment for NBIS                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CAMERA:                                                     │
│  ├── Resolution: minimum 5 megapixels                       │
│  ├── Sensor: CMOS or CCD                                    │
│  ├── Lens: fixed focal length (no zoom distortion)          │
│  ├── Auto-focus: software-controlled (locks on face)        │
│  └── Anti-red-eye: LED ring or controlled flash            │
│                                                              │
│  LIGHTING SETUP:                                             │
│  ├── Front lighting: two diffuse lights, one each side      │
│  ├── Background light: even illumination on backdrop        │
│  ├── No overhead lighting (creates shadows under eyes)     │
│  └── Color temperature: 5500K–6500K (daylight equivalent)  │
│                                                              │
│  BACKGROUND:                                                 │
│  ├── Fixed backdrop: 1.5m × 2m white/light gray fabric     │
│  ├── OR controlled background removal (software)            │
│  └── Distance from subject to backdrop: 0.5–1m             │
│                                                              │
│  POSITIONING GUIDE:                                          │
│  ├── Chin rest or head guide (ensures correct distance)    │
│  ├── Height adjustment marker (camera at eye level)        │
│  └── Screen shows subject their own preview (helps pose)   │
│                                                              │
│  DEDICATED FACE CAPTURE STATION (MOSIP-certified):          │
│  ├── Camera + lighting + backdrop integrated unit           │
│  ├── Software: MOSIP Device Specification (MDS) compliant  │
│  ├── Liveness: built-in PAD (blink detection, 3D depth)    │
│  └── Examples: Idemia VX100, NEC FR-ONE, Thales LiveScan   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.5 Face Capture Workflow

```
┌──────────────────────────────────────────────────────────────┐
│              Face Capture Step-by-Step Workflow              │
└──────────────────────────────────────────────────────────────┘

BEFORE CAPTURE:
  ├── Officer positions citizen at capture station
  ├── Citizen sits or stands at correct distance (0.5m)
  ├── Camera at eye level (adjust if subject in wheelchair)
  ├── Officer instructs:
  │   ├── "Look directly at the camera lens"
  │   ├── "Neutral expression — relax your face"
  │   ├── "Keep eyes open"
  │   └── "Remove glasses if wearing"
  └── Software shows live preview on screen

AUTOMATED PRE-CAPTURE CHECKS (real-time on preview):
  ├── Face detected: YES / NO
  ├── Multiple faces in frame: REJECT (only one allowed)
  ├── Pose check:
  │   ├── Yaw angle: ±5°? ✅ / ❌
  │   ├── Roll angle: ±5°? ✅ / ❌
  │   └── Pitch angle: ±5°? ✅ / ❌
  ├── Eye check: both eyes open? ✅ / ❌
  ├── Lighting: uniform? no shadows? ✅ / ❌
  ├── Occlusion check:
  │   ├── Mask covering face? REJECT
  │   ├── Hand in front of face? REJECT
  │   └── Hair across eyes? WARNING
  └── All checks PASS → software auto-triggers capture
      (or officer presses button)

CAPTURE:
  Camera captures image
         │
         ▼
  IMMEDIATE QUALITY ASSESSMENT:
  ├── ICAO compliance check (automated)
  ├── Face quality score (0–100)
  ├── Display captured image to officer and citizen
  └── Citizen confirms: "Is this a good photo?"

QUALITY SCORE ≥ 70: ACCEPT
  ├── Officer and citizen confirm
  └── Image stored in enrollment packet

QUALITY SCORE 50–69: WARNING
  ├── Officer sees warning with reason
  │   ("Slight head tilt detected — retake recommended")
  ├── Officer can accept or retry
  └── Policy: up to 3 retakes

QUALITY SCORE < 50: REJECT
  ├── Mandatory retake
  ├── Officer guided on what to fix:
  │   ├── "Ask citizen to open eyes wider"
  │   ├── "Adjust lighting — shadow on left side"
  │   └── "Ensure neutral expression"
  └── After 3 attempts: supervisor handles exception
```

---

## 21.6 Face Quality Metrics

Face quality is assessed across multiple dimensions, each measured independently:

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality Assessment Dimensions              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  GEOMETRIC QUALITY:                                          │
│  ├── Face size: eye-to-eye distance ≥ 90 pixels            │
│  ├── Face centrality: face centered in frame ±10%          │
│  ├── Pose angles: yaw, pitch, roll all within ±5°          │
│  └── Symmetry: both sides of face visible                  │
│                                                              │
│  EYE QUALITY:                                                │
│  ├── Both eyes detected: YES/NO                            │
│  ├── Left eye open score: 0–100                            │
│  ├── Right eye open score: 0–100                           │
│  ├── Eye gaze: looking at camera (not sideways)            │
│  └── Red-eye: detected / not detected                      │
│                                                              │
│  LIGHTING QUALITY:                                           │
│  ├── Brightness: within acceptable range (not too bright/dark)│
│  ├── Contrast: sufficient to distinguish facial features   │
│  ├── Uniformity: both sides of face equally lit            │
│  └── Shadow score: shadow coverage < 10% of face area      │
│                                                              │
│  SHARPNESS:                                                  │
│  ├── Focus score: face in sharp focus                      │
│  ├── Motion blur: no blur from movement                    │
│  └── Overall sharpness score: 0–100                        │
│                                                              │
│  OCCLUSION:                                                  │
│  ├── Glasses: present / absent                             │
│  ├── Mask: present / absent (auto-reject if present)       │
│  ├── Hair: covering eyes / not covering                    │
│  └── Hand / object in front of face: present / absent     │
│                                                              │
│  EXPRESSION:                                                 │
│  ├── Mouth open score                                       │
│  ├── Smile intensity score                                  │
│  └── Neutral expression score (target: high)               │
│                                                              │
│  COMBINED QUALITY SCORE:                                     │
│  Weighted average of all dimensions → 0–100                │
│  ├── Score ≥ 70: GOOD (accept)                             │
│  ├── Score 50–69: FAIR (warn + option to retake)          │
│  └── Score < 50: POOR (mandatory retake)                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.7 Face Template Extraction

```
┌──────────────────────────────────────────────────────────────┐
│              Face Template Extraction Pipeline               │
└──────────────────────────────────────────────────────────────┘

INPUT: ICAO-compliant face image (600×800px, JPEG/JPEG2000)
         │
         ▼
STEP 1: Face Detection
  └── Detect face bounding box in image
      (even if well-framed, always re-detect for consistency)

STEP 2: Face Alignment
  ├── Detect facial landmarks:
  │   ├── Eye corners (left outer, left inner, right inner,
  │   │               right outer)
  │   ├── Nose tip
  │   └── Mouth corners
  ├── Normalize: rotate + scale to standard alignment
  │   (eyes always at fixed coordinates in normalized image)
  └── Crop to standard face region (forehead to chin)

STEP 3: Deep Neural Network Embedding
  ├── Run aligned face through deep CNN
  │   (FaceNet, ArcFace, or equivalent)
  ├── Network outputs: 128 or 512-dimensional vector
  │   (float array representing face geometry)
  └── This vector IS the template — called face embedding

STEP 4: Template Quality Check
  ├── Vector norm within expected range?
  ├── Embedding distinct from zero vector?
  └── Quality score confirms capture was sufficient?

STEP 5: Template Encoding
  ├── Format: ISO/IEC 19794-5 (Face Image Data)
  ├── Embedding stored as binary array
  └── Size: ~1–2 KB per face template

OUTPUT:
  Face embedding (128-d or 512-d float vector)
  + ICAO-compliant photograph (for ID card printing)
         │
         ▼
  AES-256 ENCRYPTION (KMS per-citizen key)
         │
         ▼
  S3 STORAGE:
  ├── Template: s3://nbis-templates/{uinHash}/face.enc
  └── Photo:    s3://nbis-photos/{uinHash}/face-icao.enc
                (used for ID card printing — separate bucket,
                 more restricted access)
```

---

## 21.8 Face Matching — 1:1 Verification

```
┌──────────────────────────────────────────────────────────────┐
│              Face 1:1 Matching Process                       │
└──────────────────────────────────────────────────────────────┘

Authentication request (face mode):
  Citizen claims UIN: 123456789012
  Citizen presents: live face (selfie or camera capture)
         │
         ▼
Auth Service:
  1. Liveness check FIRST (before any matching)
     ├── Pass: proceed
     └── Fail: AUTH_011 LIVENESS_FAILED

  2. Extract probe embedding from live face
     └── Same CNN as enrollment extraction

  3. Fetch enrolled embedding from S3
     └── s3://nbis-templates/{uinHash}/face.enc
     └── KMS decrypt

  4. Compute similarity score:
     └── Cosine similarity between probe and enrolled vector

     cosine_similarity = (A · B) / (||A|| × ||B||)
     where A = probe embedding, B = enrolled embedding
     Result: -1 to 1 (1 = identical, 0 = unrelated)

     Convert to percentage: score = (similarity + 1) / 2 × 100

  5. Threshold decision:
     ├── score ≥ 80: MATCH ✅
     └── score < 80: NO MATCH ❌

FACE MATCHING THRESHOLDS:
  ┌──────────────────────────────────────────────────────┐
  │ Use Case              │ Threshold │ Reason           │
  ├───────────────────────┼───────────┼──────────────────┤
  │ eGate (border)        │ 90%       │ High security     │
  │ Bank eKYC             │ 80%       │ Balanced          │
  │ Resident portal       │ 75%       │ Low risk service  │
  │ Deduplication (1:N)   │ 70%       │ Cast wide net,    │
  │                       │           │ human review      │
  └──────────────────────────────────────────────────────┘
```

---

## 21.9 Liveness Detection for Face

Face liveness detection is the most actively developed area in biometrics because 2D face spoofing (holding up a photo) is trivially easy without it:

```
┌──────────────────────────────────────────────────────────────┐
│              Face Liveness Detection (PAD)                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ATTACK TYPES:                                               │
│  ├── 2D print attack: printed photo of target              │
│  ├── 2D screen attack: tablet/phone showing target photo   │
│  ├── 3D mask: realistic mask of target face                │
│  ├── Deepfake video: AI-generated video of target          │
│  └── Makeup / silicone: disguise to resemble target        │
│                                                              │
│  PASSIVE LIVENESS (no user action required):                 │
│  ├── Texture analysis:                                      │
│  │   ├── Real skin: micro-texture patterns (pores, hair)   │
│  │   ├── Printed photo: flat, regular pixel pattern        │
│  │   └── Screen: moiré pattern from display pixels         │
│  ├── Frequency domain analysis:                            │
│  │   └── Real faces have specific frequency signatures     │
│  │       that printed photos do not replicate              │
│  ├── Depth estimation:                                      │
│  │   └── Real face has 3D structure (nose protrudes,       │
│  │       eye sockets are deeper) — photo is flat           │
│  └── Deep learning classifier (CNNs trained on            │
│      thousands of genuine vs spoof examples)               │
│                                                              │
│  ACTIVE LIVENESS (user performs action):                     │
│  ├── Blink detection: "Please blink twice"                 │
│  ├── Head turn: "Turn head left, now right"                │
│  ├── Smile: "Please smile"                                  │
│  ├── Random challenge: system asks unpredictable action    │
│  └── Number display: "Say or hold up fingers showing 3"    │
│                                                              │
│  3D DEPTH SENSING (hardware):                               │
│  ├── IR dot projector (like Face ID on iPhone)             │
│  ├── Projects 30,000 invisible IR dots on face             │
│  ├── IR camera captures dot distortion → 3D depth map      │
│  └── Photo has no depth → instantly detected as spoof      │
│                                                              │
│  NBIS RECOMMENDATION:                                        │
│  ├── Enrollment: passive + active (controlled environment) │
│  ├── eGate authentication: 3D depth sensor                 │
│  ├── Remote eKYC: passive + blink detection                │
│  └── All: ISO 30107-3 PAD Level 1 minimum                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.10 Aging and Face Recognition

Unlike fingerprints (stable for life), faces change significantly with age. NBIS must account for this:

```
┌──────────────────────────────────────────────────────────────┐
│              Face Aging Impact on Recognition                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  AGE GROUP CHANGES:                                          │
│  ├── 0–5 years: face changes dramatically (do not use      │
│  │              for biometric authentication)               │
│  ├── 5–18 years: significant changes (re-enroll at 18)     │
│  ├── 18–40 years: slow change (accuracy stays high)        │
│  ├── 40–60 years: moderate change (wrinkles, weight)       │
│  └── 60+ years: significant change (sagging, hair loss)   │
│                                                              │
│  ACCURACY DEGRADATION OVER TIME:                             │
│  ├── 0–5 years since enrollment: minimal degradation       │
│  ├── 5–10 years: ~5–10% accuracy reduction                │
│  └── 10+ years: significant degradation (re-enrollment    │
│      recommended)                                           │
│                                                              │
│  NBIS POLICY FOR FACE REFRESH:                               │
│  ├── Mandatory re-enrollment: every 10 years               │
│  ├── Credential renewal triggers face update               │
│  ├── Citizen can request face update at any time           │
│  └── Age > 60: recommend 5-year refresh cycle              │
│                                                              │
│  MODERN DEEP LEARNING ADVANTAGE:                             │
│  Modern face recognition models (ArcFace, FaceNet) are     │
│  trained on age-diverse datasets and handle aging much      │
│  better than older eigenface/PCA approaches.               │
│  State-of-the-art: < 1% accuracy drop over 10 years       │
│  for adults with good enrollment quality.                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.11 Fairness and Demographic Accuracy

Face recognition accuracy is not uniform across all demographic groups. This is a well-documented engineering and ethical issue that NBIS engineers must understand:

```
┌──────────────────────────────────────────────────────────────┐
│              Face Recognition Fairness                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  DOCUMENTED DISPARITIES (NIST FRVT studies):                │
│  ├── Older algorithms showed higher FMR for:               │
│  │   ├── Darker skin tones                                  │
│  │   ├── Women vs men                                       │
│  │   └── Elderly                                           │
│  └── Modern algorithms (post-2020) significantly reduced   │
│      but not eliminated these disparities                   │
│                                                              │
│  ROOT CAUSES:                                                │
│  ├── Training data: underrepresentation of some groups     │
│  ├── Lighting: algorithms trained on light skin tones      │
│  │   may perform worse in low contrast (dark skin + low   │
│  │   quality lighting)                                      │
│  └── Image quality: same quality threshold affects        │
│      different groups differently                          │
│                                                              │
│  NBIS ENGINEERING REQUIREMENTS:                              │
│  ├── MANDATORY: Evaluate chosen face algorithm on          │
│  │   demographic subgroups of the enrolling population     │
│  ├── Test: NIST FRVT results for the specific algorithm   │
│  ├── Lighting: ensure good lighting at ALL stations       │
│  │   (especially for dark skin tones)                     │
│  ├── Threshold: potentially different thresholds per       │
│  │   demographic group (advanced — controversial)          │
│  └── Monitor: ongoing fairness metrics in production      │
│      (FRR per demographic group — alert on disparities)   │
│                                                              │
│  PRACTICAL GUIDANCE:                                         │
│  Before deploying any face algorithm:                       │
│  ├── Check NIST FRVT rankings: nvlpubs.nist.gov/nistpubs  │
│  ├── Choose algorithm with lowest demographic variance     │
│  ├── Conduct local demographic testing before go-live     │
│  └── Commit to monitoring and quarterly review            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.12 Exception Handling

```
┌──────────────────────────────────────────────────────────────┐
│              Face Capture Exception Scenarios                │
├─────────────────────────────────┬────────────────────────────┤
│ Scenario                        │ Handling                  │
├─────────────────────────────────┼────────────────────────────┤
│ Citizen wears hijab / niqab     │ Hijab: permitted          │
│                                 │ (hair not required for    │
│                                 │ face recognition)         │
│                                 │ Niqab: remove for capture │
│                                 │ (eyes/nose/mouth needed)  │
│                                 │ Private room + female     │
│                                 │ officer option provided   │
├─────────────────────────────────┼────────────────────────────┤
│ Glasses (regular)               │ Strongly recommend remove │
│                                 │ (glare obscures eyes)     │
│                                 │ If medical necessity:     │
│                                 │ document, supervisor OK   │
├─────────────────────────────────┼────────────────────────────┤
│ Glasses (dark / sunglasses)     │ Must remove — no          │
│                                 │ exceptions                │
├─────────────────────────────────┼────────────────────────────┤
│ Birthmark / facial scar         │ Do not attempt to hide    │
│                                 │ or retouch — document     │
│                                 │ natural appearance        │
├─────────────────────────────────┼────────────────────────────┤
│ Facial paralysis / drooping     │ Capture best available    │
│ (Bell's palsy, stroke)          │ face. Document condition. │
│                                 │ Flag: FACE_CONDITION      │
├─────────────────────────────────┼────────────────────────────┤
│ Newborn / infant                │ Capture face (no          │
│                                 │ fingerprints at this age).│
│                                 │ Use parent's arms to      │
│                                 │ hold in position.         │
│                                 │ Accept lower quality.     │
├─────────────────────────────────┼────────────────────────────┤
│ Blind person (eyes closed)      │ Assist with eyes-open     │
│                                 │ position. If unable:      │
│                                 │ document, supervisor OK.  │
├─────────────────────────────────┼────────────────────────────┤
│ Active skin condition           │ Capture as-is — genuine   │
│ (vitiligo, severe rash)         │ appearance must be        │
│                                 │ recorded. No editing.     │
└─────────────────────────────────┴────────────────────────────┘
```

---

## 21.13 Face Photo for ID Card Printing

The same face captured for biometric template is also used for printing on the physical ID card. These are two separate use cases with slightly different requirements:

```
┌──────────────────────────────────────────────────────────────┐
│        Face Image: Biometric vs ID Card Print                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  BIOMETRIC TEMPLATE USE:                                     │
│  ├── Resolution: 600×800 minimum for good embedding        │
│  ├── Format: JPEG 2000 (ISO 19794-5)                       │
│  ├── Color: RGB (color) for modern algorithms              │
│  ├── Grayscale: also stored (some older ABIS prefer)       │
│  ├── Storage: S3 (nbis-biometric-templates, restricted)    │
│  └── Access: biometric service only                        │
│                                                              │
│  ID CARD PRINT USE:                                          │
│  ├── Resolution: 600 DPI at print size (e.g. 25×32mm)     │
│  │   = ~590×755 pixels at 600 DPI for passport size       │
│  ├── Format: JPEG (print-ready)                            │
│  ├── Color: RGB → CMYK conversion by card vendor          │
│  ├── Storage: S3 (nbis-card-print-data, card vendor only) │
│  └── Access: credential service + card vendor only        │
│                                                              │
│  IMPORTANT:                                                  │
│  NO EDITING OR FILTERING of face photos is permitted.       │
│  ├── No beauty filters                                     │
│  ├── No skin smoothing                                     │
│  ├── No color correction on face tones                     │
│  ├── No removal of birthmarks, scars, or glasses          │
│  └── The photo must represent the person's true           │
│      natural appearance for law enforcement recognition    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.14 AWS Architecture for Face Capture Processing

```
┌──────────────────────────────────────────────────────────────┐
│              Face Capture Processing — AWS                   │
└──────────────────────────────────────────────────────────────┘

Registration Center (MDS-certified face capture station)
  │
  │ MDS-signed face image + metadata (JWS)
  │ Inside encrypted enrollment packet
  ▼
API Gateway → Registration Service (ECS)
  │
  └──► S3: store encrypted packet
       SQS: ENROLLMENT_RECEIVED event

SQS → Biometric Service (ECS Fargate)
  │
  ├──► STEP 1: Verify MDS device signature (JWS)
  │
  ├──► STEP 2: Decrypt face image (KMS)
  │
  ├──► STEP 3: ICAO compliance check
  │    └── Amazon Rekognition: DetectFaces API
  │        Returns: pose angles, eye status,
  │        brightness, sharpness, face size
  │        → All within ICAO spec? ✅ / ❌
  │
  ├──► STEP 4: Liveness detection (enrollment station PAD)
  │    └── PAD result from MDS device payload
  │        (hardware liveness built into station)
  │
  ├──► STEP 5: Face template extraction
  │    └── Deep learning model (ArcFace or equivalent)
  │        running in ECS container (GPU-optional)
  │        → 512-dimensional face embedding
  │
  ├──► STEP 6: Document cross-check (fraud detection)
  │    └── Amazon Rekognition: CompareFaces API
  │        Compare: enrolled face vs passport photo
  │        (previously OCR'd from document scan)
  │        Similarity < 70%? → FLAG for supervisor review
  │
  ├──► STEP 7: Encrypt and store
  │    ├── S3: s3://nbis-templates/{uinHash}/face.enc
  │    └── S3: s3://nbis-card-photos/{uinHash}/print.enc
  │
  ├──► STEP 8: Update enrollment status (DynamoDB)
  │
  └──► STEP 9: Publish to SQS → deduplication next

AWS SERVICES USED:
  ├── Amazon Rekognition: ICAO check + face comparison
  ├── ECS Fargate:        face extraction + processing
  ├── S3 + KMS:           encrypted template + photo storage
  ├── DynamoDB:           status tracking
  ├── SQS:                async pipeline
  └── CloudWatch:         quality metrics

REKOGNITION vs CUSTOM MODEL:
  ├── Amazon Rekognition: excellent for ICAO quality check
  │   and document cross-check (face comparison)
  ├── Custom model (ArcFace):better for biometric matching
  │   (optimized for 1:1 and 1:N identity verification)
  └── Best practice: Rekognition for QC checks,
      custom model for identity verification matching
```

---

## 21.15 Face Quality Metrics Monitoring

```
┌──────────────────────────────────────────────────────────────┐
│              Face Quality CloudWatch Metrics                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PER ENROLLMENT CENTER (daily):                              │
│  ├── average_face_quality_score                             │
│  ├── icao_compliance_rate (% passing ICAO checks)          │
│  ├── recapture_rate (% needing more than 1 attempt)        │
│  ├── liveness_fail_rate (% failing PAD)                    │
│  └── exception_rate (% needing supervisor)                 │
│                                                              │
│  ALARMS:                                                     │
│  ├── icao_compliance_rate < 80% for 1 hour                 │
│  │   → Alert: camera misconfigured or lighting issue       │
│  ├── average_face_quality < 60 for 1 hour                  │
│  │   → Alert: inspect equipment / lighting                 │
│  └── liveness_fail_rate > 5% for 30 min                   │
│      → Alert: possible spoofing attempt at this center     │
│                                                              │
│  DASHBOARD: Per-center quality heatmap                      │
│  Shows which centers are producing low-quality captures     │
│  → Targeted training / equipment inspection                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.16 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Face Capture Reference                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Client — face capture:                  │
│  ├── MDS-compliant face capture device integration          │
│  ├── Displays live preview with pose guidance overlay       │
│  ├── ICAO compliance check after capture                   │
│  ├── Shows quality score to officer                        │
│  ├── Allows up to 3 retakes per enrollment                 │
│  └── Exception documentation if all retakes fail          │
│                                                              │
│  MOSIP biometric attribute name: "face"                     │
│  (Stored in MOSIP ID Object Definition as bio attribute)   │
│                                                              │
│  MOSIP face template standard:                              │
│  └── ISO/IEC 19794-5 Face Image Data (wrapped in          │
│      CBEFF — Common Biometric Exchange Formats Framework)  │
│                                                              │
│  MOSIP face in ABIS:                                        │
│  ├── Face templates sent to ABIS alongside fingerprints    │
│  ├── ABIS performs multimodal deduplication                │
│  │   (face + fingerprint together improves accuracy)       │
│  └── For children < 5: face is the ONLY biometric          │
│      (fingerprints not captured)                           │
│                                                              │
│  MOSIP face in ID Authentication (IDA):                     │
│  ├── Face authentication supported via IDA API             │
│  ├── Requires: liveness detection result from device       │
│  ├── Matching: configurable (Rekognition or custom)       │
│  └── MOSIP does not bundle a face matching algorithm —     │
│      countries integrate their own (or Amazon Rekognition) │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 21.17 Key Terms

| Term | Definition |
|------|-----------|
| **Face capture** | Recording a standardized face photograph during enrollment |
| **ICAO Doc 9303** | International standard for face photographs in identity documents |
| **Face embedding** | Deep learning vector (128 or 512 dimensions) representing face geometry |
| **Cosine similarity** | Mathematical measure of similarity between two face embeddings |
| **ArcFace** | State-of-the-art deep learning face recognition algorithm |
| **FaceNet** | Google's face recognition algorithm producing 128-d embeddings |
| **Pose angles** | Yaw (left-right), pitch (up-down), roll (tilt) — all ≤ ±5° for ICAO |
| **PAD** | Presentation Attack Detection — liveness detection for face |
| **2D print attack** | Holding a printed photo in front of camera to spoof face recognition |
| **3D mask attack** | Using a 3D-printed or sculptured mask of a person's face |
| **Deepfake** | AI-generated synthetic video of a person's face |
| **Passive liveness** | Liveness detection without requiring user action |
| **Active liveness** | Liveness detection that asks user to perform an action (blink, turn) |
| **IR dot projector** | Hardware for 3D depth sensing — projects infrared dots onto face |
| **ISO 19794-5** | International standard for face image data format |
| **CBEFF** | Common Biometric Exchange Formats Framework — wraps biometric templates |
| **FRVT** | Face Recognition Vendor Testing — NIST program evaluating face algorithms |
| **NIST** | National Institute of Standards and Technology — sets biometric standards |
| **Face aging** | Natural change in facial appearance over time affecting recognition |
| **Demographic variance** | Difference in face recognition accuracy across demographic groups |
| **Yaw** | Left-right head rotation angle |
| **Pitch** | Up-down head rotation angle (chin up/down) |
| **Roll** | Head tilt (ear toward shoulder) angle |

---

## 21.18 Key Takeaways

- **Face has three roles in NBIS simultaneously** — biometric template, ID card photo, and document cross-check tool. Capture quality matters for all three.
- **ICAO compliance is non-negotiable** — the enrolled face photo travels with the citizen internationally. Non-compliant photos cause border system failures.
- **Liveness detection is the most critical security control for face** — 2D photo spoofing requires nothing more than a printer. Without PAD, face authentication is trivially bypassable.
- **No editing of face photos is ever permitted** — the photo must represent the person's true, unmodified appearance. Filters, retouching, or color adjustments are prohibited for law enforcement and border recognition reasons.
- **Face degrades faster than fingerprints** — plan for 10-year re-enrollment cycles for adults and 5-year cycles for children and elderly. Build this into the lifecycle management system.
- **Demographic fairness testing is mandatory before go-live** — check NIST FRVT rankings for the algorithm you choose. Test on a sample of your actual enrolling population. Do not assume algorithms tested in the USA perform equally across all demographics in your country.
- **Amazon Rekognition for quality checks, custom model for matching** — Rekognition is excellent for ICAO compliance checking and document comparison. For 1:1 and 1:N biometric matching, a dedicated identity-grade model (ArcFace, CosFace) gives better accuracy.
- **The face photo doubles as the ID card image** — capture at sufficient resolution (600×800 minimum) for both biometric matching and print quality. A single high-quality capture serves both purposes.

---

## 21.19 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 22 | Iris Capture — NIR camera, IrisCode, quality metrics, exceptions |
| Chapter 23 | Signature Capture — digital pad, format, use cases |
| Chapter 24 | Quality Checks — overall quality pipeline, rejection handling |

---

*Chapter 21 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
