<invoke name="create_file">
<parameter name="description">Chapter 22 - Iris Capture</parameter>
<parameter name="file_text"># Chapter 22 — Iris Capture

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 22.1 What is Iris Capture?

**Iris capture** is the process of recording a high-resolution photograph of the colored ring of the eye — the iris — using a Near-Infrared (NIR) camera during enrollment. The unique texture pattern of the iris is then converted into an **IrisCode** for storage and matching.

The iris is widely regarded as the **most accurate biometric modality** in national identity systems. It combines extraordinary uniqueness with lifetime stability — a person's iris pattern does not change from early childhood to death.

```
┌──────────────────────────────────────────────────────────────┐
│              Why Iris Is the Gold Standard                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Uniqueness:   Probability of two irises matching by      │
│                  chance: 1 in 10^78                          │
│                  (Even identical twins have different irises)│
│                                                              │
│  ✅ Stability:   Iris pattern set by age 2 and does NOT     │
│                  change. Ever.                               │
│                  (Unlike face: no aging impact)              │
│                                                              │
│  ✅ Accuracy:    EER as low as 0.0001% in ideal conditions  │
│                  (100× more accurate than fingerprint)       │
│                                                              │
│  ✅ Non-contact: Camera captures at 20–40cm distance        │
│                  No touching, no discomfort                  │
│                                                              │
│  ✅ Fast:        Capture in < 3 seconds per eye             │
│                  Authentication in < 1 second               │
│                                                              │
│  ⚠️  Challenges:                                             │
│  ├── NIR camera required (specialized, more expensive)      │
│  ├── Cataracts / eye surgery can affect iris texture        │
│  ├── Prosthetic eyes cannot be captured                     │
│  ├── Very dark irises: lower contrast in NIR               │
│  └── Requires cooperation (must look at camera)             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.2 Iris Anatomy — What Makes It Unique

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Anatomy                                    │
└──────────────────────────────────────────────────────────────┘

STRUCTURE OF THE IRIS:

  ┌──────────────────────────────────────────────────────┐
  │                   WHITE (Sclera)                     │
  │   ┌──────────────────────────────────────────────┐   │
  │   │              IRIS (colored ring)             │   │
  │   │   ┌──────────────────────────────────────┐  │   │
  │   │   │           PUPIL (black center)       │  │   │
  │   │   └──────────────────────────────────────┘  │   │
  │   │                                              │   │
  │   │  Features that make it unique:               │   │
  │   │  ├── Crypts (pits in iris tissue)           │   │
  │   │  ├── Furrows (concentric wrinkles)           │   │
  │   │  ├── Radial striations (spoke-like lines)   │   │
  │   │  ├── Collarette (inner border of iris)      │   │
  │   │  └── Pigment spots (darker patches)         │   │
  │   └──────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘

KEY FACTS:
  ├── Iris area: ~1 cm² (small but extremely complex)
  ├── Degrees of freedom: ~249 independent features
  │   (fingerprint: ~40–100 minutiae points by comparison)
  ├── Development: complete by age 2
  ├── Stability: permanent (no aging, no change from injury
  │              unless iris itself is physically damaged)
  └── Left vs Right: completely different patterns
      (two irises per person = two independent biometrics)
```

---

## 22.3 Near-Infrared (NIR) Imaging

Iris capture uses **Near-Infrared (NIR) light** — invisible to the human eye — rather than visible light. This is a critical design choice:

```
┌──────────────────────────────────────────────────────────────┐
│              Why NIR for Iris Capture                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  VISIBLE LIGHT PROBLEM:                                      │
│  Dark brown irises (common in Middle East, South Asia,      │
│  Africa) absorb visible light heavily.                       │
│  ├── Brown iris under visible light: mostly featureless    │
│  ├── Texture patterns hidden by melanin pigment            │
│  └── Recognition accuracy: very low for dark irises        │
│                                                              │
│  NIR SOLUTION:                                               │
│  Near-infrared light (700–900nm wavelength) penetrates      │
│  the melanin layer and reveals the underlying iris          │
│  texture regardless of iris color.                           │
│                                                              │
│  NIR result:                                                 │
│  ├── Brown iris under NIR: rich texture visible            │
│  ├── Blue iris under NIR: texture visible                  │
│  ├── Green iris under NIR: texture visible                 │
│  └── All iris colors: equally high accuracy with NIR       │
│                                                              │
│  ADDITIONAL NIR BENEFITS:                                    │
│  ├── Pupil dilation consistent under NIR (no bright flash) │
│  ├── Ambient light variation matters less                   │
│  └── Invisible to subject (no discomfort from flash)       │
│                                                              │
│  WAVELENGTH STANDARD:                                        │
│  ISO/IEC 19794-6: 700–900nm NIR illumination               │
│  Most iris cameras use 850nm LEDs                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.4 Iris Capture Equipment

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Capture Equipment for NBIS                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  IRIS CAMERA TYPES:                                          │
│                                                              │
│  1. SINGLE-EYE CAPTURE (contact distance, 5–40cm)           │
│  ├── One eye at a time                                      │
│  ├── High resolution: 640×480 minimum (iris region)        │
│  ├── NIR LEDs: ring illumination around lens               │
│  ├── Auto-focus: lens adjusts for optimal iris sharpness   │
│  ├── Distance sensor: guides subject to optimal range      │
│  └── Examples: IriShield, Iris ID, Princeton Identity      │
│                                                              │
│  2. DUAL-EYE CAPTURE (simultaneous, 20–50cm)                │
│  ├── Both irises captured simultaneously                   │
│  ├── Faster enrollment (single interaction)                │
│  ├── More expensive than single-eye                        │
│  ├── Wider field of view requires high-res sensor          │
│  └── Examples: Iris ID iCAM TD100, Idemia IriStar         │
│                                                              │
│  3. AT-A-DISTANCE / ON-THE-MOVE (0.5–3m)                    │
│  ├── Long-range iris capture                               │
│  ├── High-powered NIR + telephoto lens                     │
│  ├── Used at border crossings (no stopping required)       │
│  ├── Very expensive (USD 30,000+)                          │
│  └── Examples: AOptix, SRI Iris on the Move (IOM)         │
│                                                              │
│  MINIMUM SPECIFICATIONS (NBIS enrollment):                   │
│  ├── Resolution: 640×480 pixels (iris region only)         │
│  ├── NIR wavelength: 700–900nm                             │
│  ├── Capture distance: 15–35cm                             │
│  ├── Auto-focus: < 1 second lock time                      │
│  ├── PAD (liveness): required                              │
│  └── Certification: MOSIP MDS compliant                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.5 IrisCode — The Iris Template

The **IrisCode** is the mathematical representation of an iris texture pattern, developed by Dr. John Daugman (Cambridge University) and the basis of virtually every commercial iris recognition system worldwide.

```
┌──────────────────────────────────────────────────────────────┐
│              IrisCode Generation                             │
└──────────────────────────────────────────────────────────────┘

STEP 1: IRIS SEGMENTATION
  ├── Locate pupil boundary (inner circle)
  ├── Locate limbus boundary (outer iris edge)
  ├── Exclude eyelid regions (top and bottom)
  ├── Exclude eyelash regions
  └── Extract: usable iris annular region

  Usable iris area varies per person:
  ├── Wide-open eyes → more iris visible → better quality
  └── Half-closed eyes → eyelids occlude iris → less usable

STEP 2: NORMALIZATION (Daugman rubber sheet model)
  ├── Unwrap iris from circular to rectangular strip
  ├── Maps (r, θ) polar coordinates →
  │   rectangular grid of fixed size (64×512 standard)
  ├── Compensates for pupil dilation variation
  │   (pupil expands/contracts with lighting —
  │    normalization stretches/compresses to match)
  └── Output: 64×512 grayscale iris strip

STEP 3: FEATURE EXTRACTION (Gabor wavelets)
  ├── Apply 2D Gabor filters at multiple scales
  │   and orientations to the normalized iris strip
  ├── Each filter responds to specific texture patterns
  └── Each filter output → binarized:
      response > 0 → bit = 1
      response < 0 → bit = 0

STEP 4: IRISCODE GENERATION
  ├── Result: 2048 bits (256 bytes) of texture pattern
  ├── Plus: 2048-bit mask (which bits are reliable)
  │   (Mask = 0 where eyelids/lashes occluded iris)
  └── IrisCode = 2048 data bits + 2048 mask bits
                = 4096 bits (512 bytes) total

  IrisCode example (first 32 bits):
  10110001 01110100 11001010 00110111 ...

PROPERTIES:
  ├── Size: 512 bytes per iris (tiny)
  ├── Comparison: Hamming distance (bitwise XOR)
  ├── Two irises from same person:
  │   Hamming distance ≈ 0.08–0.12 (very similar)
  └── Two irises from different people:
      Hamming distance ≈ 0.45–0.50 (near random)
      → Clearly separable distributions
```

---

## 22.6 Iris Quality Metrics

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Quality Assessment (ISO/IEC 29794-6)       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  QUALITY DIMENSIONS:                                         │
│                                                              │
│  1. USABLE IRIS AREA (most important)                        │
│  Percentage of iris not occluded by eyelids/eyelashes.      │
│  ├── Target: ≥ 70% usable                                  │
│  ├── Minimum: ≥ 50% usable                                 │
│  └── < 50%: insufficient data for reliable IrisCode        │
│                                                              │
│  2. IRIS SHARPNESS (focus)                                   │
│  Is the iris in sharp focus?                                │
│  ├── Measured: frequency content of iris region            │
│  ├── Blurry iris: low frequency → poor texture detail      │
│  └── Target: sharpness score ≥ 70 / 100                   │
│                                                              │
│  3. PUPIL DILATION RATIO                                     │
│  Ratio of pupil diameter to iris diameter.                  │
│  ├── Target: 0.2 – 0.7 (pupil not too large/small)        │
│  ├── Too dilated (> 0.7): pupil covers most of iris        │
│  └── Too constricted (< 0.2): unusual, may indicate issue  │
│                                                              │
│  4. IRIS-SCLERA CONTRAST                                     │
│  Contrast between iris and white of eye.                    │
│  ├── Low contrast → harder to locate iris boundary         │
│  └── Target: contrast score ≥ 50 / 100                    │
│                                                              │
│  5. GAZE DIRECTION                                           │
│  Is the iris centered (subject looking at camera)?          │
│  ├── Off-center iris: partial view only                    │
│  └── Target: < 10% off-center from camera axis            │
│                                                              │
│  6. MOTION BLUR                                              │
│  Subject movement during capture.                           │
│  ├── Blur smears iris texture → unusable IrisCode          │
│  └── Target: blur score < 10 / 100                        │
│                                                              │
│  OVERALL QUALITY SCORE:                                      │
│  Weighted combination of above → 0–100                     │
│  ├── ≥ 70: GOOD (accept)                                   │
│  ├── 50–69: FAIR (warn + optional retry)                   │
│  └── < 50: POOR (mandatory retry)                         │
│                                                              │
│  NBIS THRESHOLD POLICY:                                      │
│  Hard minimum: Quality ≥ 50                                 │
│  Target:       Quality ≥ 70                                 │
│  Exception path: < 50 after 3 attempts → supervisor        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.7 Iris Capture Workflow

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Capture Step-by-Step Workflow              │
└──────────────────────────────────────────────────────────────┘

BEFORE CAPTURE:
  ├── Officer positions citizen at iris camera station
  ├── Correct distance: 20–35cm from camera (varies by model)
  ├── Camera at eye level (adjust for height)
  ├── Officer instructs:
  │   ├── "Look at the green dot / mirror / light on camera"
  │   ├── "Open your eyes wide"
  │   ├── "Try not to blink"
  │   └── "Keep your head still"
  └── Camera shows: live video preview with eye tracking

AUTOMATED CAPTURE SEQUENCE:
         │
         ▼
  Eye tracking: locate eyes in frame
         │
         ▼
  Distance check: subject at correct range?
  ├── Too close: "Please move back slightly"
  └── Too far:   "Please move closer"
         │
         ▼
  Focus lock: auto-focus sharpens on iris
         │
         ▼
  Quality assessment (real-time on live frames):
  ├── Usable iris area check (≥ 50%)
  ├── Sharpness check
  ├── Gaze direction check
  ├── Motion blur check
  └── All pass → capture triggered automatically
         │
         ▼
  Capture: NIR image captured (single frame or best of burst)
         │
         ▼
  Quality score displayed: LEFT EYE: 87/100  RIGHT EYE: 82/100

SEQUENCE:
  Step 1: Left iris captured
  Step 2: Right iris captured
  Step 3: Quality summary shown to officer
  Step 4: Officer confirms or requests retry

COMMON CAPTURE ISSUES AND GUIDANCE:
  ├── Eyelids drooping: "Open your eyes wider please"
  ├── Eyelashes in frame: "Look straight ahead"
  ├── Subject blinking: wait + retry auto-trigger
  ├── Glasses: remove contact lenses if tinted
  │   (clear contact lenses OK for most systems)
  ├── Gaze off-center: "Look at the dot/light"
  └── Motion blur: "Please hold still for a moment"
```

---

## 22.8 Iris Matching — Hamming Distance

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Matching Using Hamming Distance            │
└──────────────────────────────────────────────────────────────┘

MATCHING ALGORITHM:
  Probe IrisCode:   10110001 01110100 ...  (2048 bits)
  Enrolled IrisCode:10110011 01110100 ...  (2048 bits)
         │
         ▼
  XOR:              00000010 00000000 ...
  (bits where they differ = 1)
         │
         ▼
  Apply combined mask:
  (exclude bits where EITHER code has mask = 0)
  Masked bits removed from comparison
         │
         ▼
  Count remaining differing bits
  Hamming Distance (HD) = differing bits / total unmasked bits
         │
  HD = 0.09 → MATCH ✅ (same iris, different capture)
  HD = 0.47 → NO MATCH ❌ (different iris)

ROTATIONAL COMPENSATION:
  Problem: subject's head may be slightly tilted differently
  at authentication vs enrollment → IrisCode rotated
  Solution: shift IrisCode ±n bit positions (simulate rotation)
  Match at EACH rotation position
  Take MINIMUM Hamming distance across all shifts
  → Best alignment = true Hamming distance

THRESHOLD:
  ├── HD < 0.30: MATCH (same iris with high confidence)
  ├── HD ≥ 0.30: NO MATCH (different iris)
  └── Decision zone HD 0.28–0.32: borderline (review)

  Note: HD ≈ 0.50 = completely random (coin flip)
  The range 0.30–0.50 is clearly "not a match"
  The range 0.00–0.30 is clearly "a match"
  The separation between genuine and impostor
  distributions is very wide — that is iris's great strength

NBIS THRESHOLD BY USE CASE:
  ┌──────────────────────────────────────────────────────┐
  │ Use Case          │ HD Threshold │ Equivalent FAR   │
  ├───────────────────┼──────────────┼──────────────────┤
  │ Enrollment dedup  │ 0.32         │ 1 in 10,000      │
  │ Border eGate      │ 0.28         │ 1 in 1,000,000   │
  │ Bank eKYC         │ 0.30         │ 1 in 100,000     │
  │ Resident portal   │ 0.32         │ 1 in 10,000      │
  └──────────────────────────────────────────────────────┘
```

---

## 22.9 Iris Liveness Detection

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Liveness Detection (PAD)                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ATTACK TYPES:                                               │
│  ├── Printed iris photo (high-resolution print of iris)    │
│  ├── Screen replay (iris displayed on tablet/phone)        │
│  ├── Cosmetic contact lens (pattern printed on lens)       │
│  ├── Glass eye / prosthetic (realistic artificial eye)     │
│  └── Dead eye (post-mortem — iris relaxes, pupil dilates)  │
│                                                              │
│  LIVENESS DETECTION METHODS:                                 │
│                                                              │
│  PUPIL DILATION RESPONSE:                                    │
│  ├── Flash NIR light → pupil constricts (live eye)         │
│  ├── Remove light → pupil dilates (live eye)               │
│  ├── Printed photo / glass eye: no pupil response          │
│  └── Test: measure pupil size before and after light pulse │
│                                                              │
│  TEXTURE FREQUENCY ANALYSIS:                                 │
│  ├── Real iris: complex, continuous texture                │
│  ├── Printed iris: regular dot/halftone pattern            │
│  ├── Screen replay: pixel grid pattern                     │
│  └── Contact lens: artificial pattern overlaid on real     │
│      (different frequency signature)                        │
│                                                              │
│  SPECULAR REFLECTION:                                        │
│  ├── Live eye: single specular reflection on moist cornea  │
│  ├── Printed photo: no reflection (flat surface)           │
│  ├── Screen: different reflection pattern (display glass)  │
│  └── Analysis: detect and classify reflection type         │
│                                                              │
│  VASCULAR PATTERN (advanced):                                │
│  ├── Blood vessels visible in sclera under NIR              │
│  ├── Live eye: subtle vascular pulsation detectable        │
│  └── Photo: no vascular movement                           │
│                                                              │
│  MOSIP/NBIS REQUIREMENT:                                     │
│  ISO 30107-3 PAD Level 1 minimum for iris sensors.         │
│  High-security deployments: PAD Level 2.                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.10 Iris Image Standards

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Image and Template Standards               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CAPTURE IMAGE:                                              │
│  ├── Format: RAW NIR grayscale (8-bit)                     │
│  ├── Resolution: 640×480 minimum (iris region ≥ 200px dia) │
│  ├── Color: grayscale (NIR = no color information)         │
│  └── Compression: lossless during capture                  │
│                                                              │
│  STORED IMAGE FORMAT:                                        │
│  ├── Standard: ISO/IEC 19794-6 (Iris Image Data)          │
│  ├── Compression: JPEG 2000 (lossless recommended)         │
│  ├── Size: ~15–30 KB per eye image                        │
│  └── Stored in: S3 + KMS (optional — template preferred)  │
│                                                              │
│  STORED TEMPLATE FORMAT:                                     │
│  ├── Standard: ISO/IEC 19794-6 (IrisCode format)          │
│  ├── Size: 512 bytes per iris (4096 bits)                  │
│  │   + 512 bytes mask = 1 KB per iris                     │
│  ├── Both left AND right stored separately                 │
│  └── Total per person: 2 KB (both irises)                 │
│                                                              │
│  WHAT GETS STORED IN NBIS:                                   │
│  ├── IrisCode template (always): S3 + KMS                 │
│  ├── Iris image (optional): S3 + KMS                      │
│  │   (kept for quality reassessment on upgrades)          │
│  └── Raw NIR capture: NEVER stored (no value, large)      │
│                                                              │
│  CBEFF WRAPPING:                                             │
│  └── MOSIP wraps iris templates in CBEFF format            │
│      (Common Biometric Exchange Formats Framework)         │
│      for interoperability with ABIS vendors               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.11 Exception Handling

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Capture Exception Scenarios                │
├─────────────────────────────────┬────────────────────────────┤
│ Scenario                        │ Handling                  │
├─────────────────────────────────┼────────────────────────────┤
│ Cataract (clouding of lens)     │ Iris texture may still    │
│                                 │ be partially visible.     │
│                                 │ Attempt capture.          │
│                                 │ If quality < 50: flag     │
│                                 │ CATARACT_EXCEPTION.       │
│                                 │ Use fingerprint primary.  │
├─────────────────────────────────┼────────────────────────────┤
│ Prosthetic eye (glass eye)      │ Only 1 iris capturable.  │
│                                 │ Capture the real eye.     │
│                                 │ Flag: LEFT/RIGHT_PROSTHETIC│
│                                 │ Dedup uses 1 iris only.   │
├─────────────────────────────────┼────────────────────────────┤
│ Bilateral blindness / no        │ Attempt capture.          │
│ functional irises               │ If genuinely not          │
│                                 │ capturable: medical cert. │
│                                 │ Flag: NO_IRIS_CAPTURE.    │
│                                 │ Fingerprint becomes       │
│                                 │ primary modality.         │
├─────────────────────────────────┼────────────────────────────┤
│ Tinted contact lenses           │ Must remove before        │
│                                 │ capture. Tinted lenses    │
│                                 │ block NIR, mask pattern.  │
│                                 │ Clear lenses: OK          │
│                                 │ (most systems capture     │
│                                 │ through clear lenses)     │
├─────────────────────────────────┼────────────────────────────┤
│ Nystagmus (involuntary          │ Use burst capture mode.   │
│ eye movement)                   │ Select best frame from    │
│                                 │ rapid sequence.           │
│                                 │ Supervisor approval.      │
├─────────────────────────────────┼────────────────────────────┤
│ Post-LASIK surgery              │ Iris pattern unchanged.   │
│                                 │ Corneal shape change does │
│                                 │ not affect iris texture.  │
│                                 │ Capture normally.         │
├─────────────────────────────────┼────────────────────────────┤
│ Very dark irises (low NIR       │ Adjust NIR intensity.    │
│ contrast)                       │ Use longer exposure.      │
│                                 │ Modern cameras handle     │
│                                 │ dark irises well at 850nm.│
├─────────────────────────────────┼────────────────────────────┤
│ Ptosis (drooping eyelid)        │ Position camera lower.    │
│                                 │ Ask citizen to try to     │
│                                 │ open eye wider.           │
│                                 │ Accept best available.    │
│                                 │ Flag: PTOSIS_EXCEPTION    │
└─────────────────────────────────┴────────────────────────────┘

EXCEPTION DATA IN PACKET:
{
  "biometricExceptions": [{
    "modality":      "IRIS",
    "eye":           "LEFT",
    "reason":        "PROSTHETIC_EYE",
    "documentedBy":  "OFR-007",
    "supervisorApproval": "SUP-001"
  }]
}
```

---

## 22.12 Iris Capture vs Other Modalities — Decision Guide

```
┌──────────────────────────────────────────────────────────────┐
│         When to Use Iris as Primary vs Secondary             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  IRIS AS PRIMARY DEDUP MODALITY:                             │
│  Use when:                                                   │
│  ├── Fingerprint quality consistently low for population    │
│  │   (high proportion of manual workers, elderly)          │
│  ├── Population has high proportion of amputees            │
│  └── Very high accuracy requirement (large registry)       │
│                                                              │
│  IRIS AS SECONDARY (fingerprint primary):                    │
│  Most common in NBIS deployments:                           │
│  ├── Fingerprint: primary for dedup + authentication       │
│  └── Iris: backup + multimodal for higher accuracy         │
│                                                              │
│  MULTIMODAL (iris + fingerprint):                            │
│  ├── Both captured at enrollment                           │
│  ├── ABIS uses both for deduplication (higher accuracy)    │
│  ├── Authentication: can use either modality               │
│  └── Best of both: finger damaged → use iris, and vice     │
│                                                              │
│  ACCURACY COMPARISON (EER):                                  │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Modality          │ Typical EER     │ Notes        │    │
│  ├───────────────────┼─────────────────┼──────────────┤    │
│  │ Iris (single)     │ 0.0001%         │ Best single  │    │
│  │ Fingerprint (best)│ 0.01%           │ Mature, fast │    │
│  │ Face              │ 0.1%            │ Varies widely│    │
│  │ Iris + Fingerprint│ < 0.00001%      │ Multimodal   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.13 Real-World NBIS Iris Deployments

```
┌──────────────────────────────────────────────────────────────┐
│              Iris in Real NBIS Deployments                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  INDIA (Aadhaar):                                           │
│  ├── Both irises captured at enrollment                     │
│  ├── 1.4 billion iris records (world's largest)            │
│  ├── Used for: deduplication + authentication              │
│  └── Particularly important for elderly and laborers       │
│      whose fingerprints are poor quality                   │
│                                                              │
│  UAE (Emirates ID):                                         │
│  ├── Iris used at all major border points                  │
│  ├── eGates: iris recognition in < 3 seconds              │
│  └── Highest throughput iris system globally               │
│                                                              │
│  BAHRAIN (CPR):                                             │
│  ├── Iris part of multimodal enrollment                    │
│  ├── Used alongside fingerprint for deduplication          │
│  └── Border systems use iris for eGate access             │
│                                                              │
│  JORDAN (National ID):                                      │
│  ├── Iris used for refugee enrollment (UNHCR partnership)  │
│  └── Critical for populations with worn fingerprints      │
│      from harsh outdoor labor conditions                   │
│                                                              │
│  MOSIP DEPLOYMENTS:                                         │
│  ├── Philippines: iris enrolled alongside fingerprint      │
│  ├── Morocco: iris as part of multimodal enrollment        │
│  └── Sri Lanka: iris for high-assurance authentication    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.14 AWS Architecture for Iris Capture Processing

```
┌──────────────────────────────────────────────────────────────┐
│              Iris Capture Processing — AWS                   │
└──────────────────────────────────────────────────────────────┘

Registration Center (MDS-certified iris camera)
  │
  │ MDS-signed NIR iris image (JWS in enrollment packet)
  ▼
API Gateway → Registration Service (ECS)
  │
  └──► S3: encrypted packet stored
       SQS: ENROLLMENT_RECEIVED event

SQS → Biometric Service (ECS Fargate)
  │
  ├──► STEP 1: Verify MDS device signature (JWS)
  │
  ├──► STEP 2: Decrypt iris NIR image (KMS)
  │
  ├──► STEP 3: Liveness check result
  │    └── PAD result from MDS device payload
  │        (hardware liveness: pupil dilation test)
  │
  ├──► STEP 4: Iris quality assessment
  │    └── Quality SDK (ISO 29794-6)
  │        running in ECS container:
  │        ├── Usable iris area calculation
  │        ├── Sharpness measurement
  │        ├── Pupil dilation ratio
  │        └── Overall quality score
  │        Quality < 50? → DLQ + notify officer for retry
  │
  ├──► STEP 5: Iris segmentation
  │    └── Locate pupil + limbus boundaries
  │        Exclude eyelid + eyelash regions
  │
  ├──► STEP 6: IrisCode generation
  │    └── Daugman rubber sheet normalization
  │        + Gabor wavelet feature extraction
  │        → 2048-bit IrisCode + 2048-bit mask
  │
  ├──► STEP 7: Encrypt IrisCode (KMS — per-citizen key)
  │
  ├──► STEP 8: Store in S3
  │    ├── s3://nbis-templates/{uinHash}/iris-left.enc
  │    └── s3://nbis-templates/{uinHash}/iris-right.enc
  │
  ├──► STEP 9: Update enrollment status (DynamoDB)
  │    └── IRIS_CAPTURE_COMPLETE
  │
  └──► STEP 10: Publish to SQS
       └── nbis-abis-requests (deduplication)

ABIS DEDUPLICATION WITH IRIS:
  ├── ABIS receives: left IrisCode + right IrisCode +
  │                  fingerprint templates (multimodal)
  ├── ABIS performs: 1:N matching for each modality
  ├── Fusion score: weighted combination of
  │   iris HD score + fingerprint match score
  └── Combined dedup: more accurate than single modality

AWS SERVICES:
  ├── ECS Fargate: iris processing container
  ├── S3 + KMS:   IrisCode + image storage
  ├── DynamoDB:   status tracking + exception records
  ├── SQS:        async pipeline
  └── CloudWatch: quality metrics

CLOUDWATCH METRICS (iris-specific):
  ├── average_iris_quality_score (per center, per day)
  ├── left_right_quality_comparison
  │   (flagged if consistently large difference)
  ├── usable_iris_area_average
  ├── liveness_fail_rate
  └── exception_rate (% requiring supervisor)
```

---

## 22.15 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Iris Capture Reference                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP Registration Client — iris capture:                  │
│  ├── MDS-compliant iris device integration                  │
│  ├── Separate capture for left and right iris              │
│  ├── Quality score displayed after each capture            │
│  ├── Up to 3 retakes per eye                               │
│  └── Exception documentation in Registration Client UI     │
│                                                              │
│  MOSIP biometric attribute names:                           │
│  ├── "leftEye"  → left iris                               │
│  └── "rightEye" → right iris                              │
│                                                              │
│  MOSIP iris standard:                                       │
│  └── ISO/IEC 19794-6 wrapped in CBEFF                     │
│                                                              │
│  ABIS integration with iris:                                │
│  ├── MOSIP sends: CBEFF-wrapped IrisCodes to ABIS queue   │
│  ├── ABIS uses iris for:                                   │
│  │   ├── 1:N deduplication (alongside fingerprint)        │
│  │   └── Multimodal fusion for higher accuracy            │
│  └── Iris authentication via IDA:                         │
│      POST /auth with bioType = IRIS                        │
│                                                              │
│  MOSIP iris exception handling:                             │
│  ├── Operator selects exception reason in UI               │
│  ├── Exception stored in enrollment packet                 │
│  └── ABIS deduplication skips iris for that candidate     │
│      if IRIS_EXCEPTION flagged                             │
│                                                              │
│  MOSIP-certified iris device vendors:                       │
│  ├── IriShield (MK2120U, BK2121)                          │
│  ├── Iris ID (iCAM series)                                 │
│  ├── Mantra (MIS100V2)                                     │
│  └── Idemia (IriStar range)                                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 22.16 Key Terms

| Term | Definition |
|------|-----------|
| **Iris capture** | Recording NIR photograph of iris for biometric enrollment |
| **IrisCode** | 2048-bit binary representation of iris texture (Daugman algorithm) |
| **NIR** | Near-Infrared — light at 700–900nm wavelength used for iris imaging |
| **Hamming distance** | Measure of difference between two IrisCodes — fraction of differing bits |
| **Segmentation** | Locating and isolating the iris region from the eye image |
| **Daugman rubber sheet** | Model for normalizing iris from circular to rectangular coordinate system |
| **Gabor wavelet** | Mathematical filter used to extract iris texture features |
| **Pupil dilation ratio** | Ratio of pupil diameter to iris diameter — quality metric |
| **Usable iris area** | Percentage of iris not occluded by eyelids or eyelashes |
| **ISO 19794-6** | International standard for iris image data format |
| **ISO 29794-6** | International standard for iris image quality metrics |
| **CBEFF** | Common Biometric Exchange Formats Framework — wraps biometric templates |
| **PAD** | Presentation Attack Detection — iris liveness detection |
| **Specular reflection** | Light reflection off the moist cornea — used for liveness detection |
| **Nystagmus** | Involuntary rapid eye movement — iris capture exception scenario |
| **Ptosis** | Drooping eyelid — reduces usable iris area |
| **Limbus** | Outer boundary between iris and white of eye (sclera) |
| **Collarette** | Inner boundary structure of the iris |
| **EER** | Equal Error Rate — iris EER ~0.0001%, much better than fingerprint |
| **Multimodal** | Using multiple biometric modalities together for higher accuracy |
| **850nm** | Standard NIR wavelength for iris cameras |

---

## 22.17 Key Takeaways

- **Iris is the most accurate single biometric modality** — EER of 0.0001% vs fingerprint's 0.01%. The Hamming distance distributions for genuine pairs and impostor pairs are cleanly separated with almost no overlap.
- **NIR imaging is mandatory — not optional** — visible light cameras cannot reliably image dark irises, which represent the majority of the Middle Eastern, South Asian, and African populations that NBIS systems serve most. NIR at 850nm gives equal accuracy across all iris colors.
- **IrisCode is tiny and permanent** — 512 bytes per iris, never changes after age 2, never degrades with aging. This makes it the perfect long-term biometric for a national identity system.
- **Usable iris area is the most critical quality metric** — if eyelids occlude more than 50% of the iris, the IrisCode does not have enough data for reliable matching. Ask citizens to open their eyes wide, and use burst capture to get the optimal frame.
- **Tinted contact lenses must be removed** — they block NIR and overlay artificial patterns. Clear contact lenses are generally acceptable. This must be in the operator's pre-capture checklist.
- **Post-LASIK and post-cataract surgery: iris pattern is unaffected** — LASIK reshapes the cornea, not the iris. IrisCode remains valid. However, cataracts can reduce image quality — attempt capture and document if below threshold.
- **Multimodal (iris + fingerprint) beats either alone** — the ABIS fusion score combining iris Hamming distance and fingerprint match score achieves EER approaching 0.00001%. For a national system with millions of records, this dramatically reduces false deduplication flags.
- **The Hamming distance decision boundary is at HD = 0.30** — below this, it is the same iris (MATCH). Above 0.35, it is definitively a different iris. The narrow gray zone between 0.30–0.35 represents borderline cases requiring supervisor review.

---

## 22.18 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 23 | Signature Capture — digital pad, format, use cases |
| Chapter 24 | Quality Checks — overall quality pipeline, rejection handling |
| Chapter 25 | Enrollment Validation — packet validation, data completeness |

---

*Chapter 22 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
</parameter>
<parameter name="path">/mnt/user-data/outputs/22-iris-capture.md</parameter>
