# Chapter 19 — Document Verification

> **Part III — Enrollment**
> `Status: Complete` | `Author: Hamza Rafique`

---

## 19.1 What is Document Verification?

**Document verification** is the process of establishing that the supporting documents presented by an applicant during enrollment are genuine, unaltered, and sufficient to prove the legal identity they claim.

It is the bridge between the civil registry (legal identity) and the NBIS (digital identity). Before biometrics are captured, the system must confirm: **"Does this person legally exist, and are they who they say they are?"**

```
┌──────────────────────────────────────────────────────────────┐
│              Why Document Verification Matters               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Without document verification:                              │
│  Anyone can walk in, claim any identity, get a UIN.         │
│  ├── Fraudsters obtain IDs in dead people's names          │
│  ├── Criminals create synthetic identities                  │
│  ├── Multiple genuine people enroll with wrong data        │
│  └── National registry filled with inaccurate records      │
│                                                              │
│  With rigorous document verification:                        │
│  ├── Identity claims are anchored to legal documents       │
│  ├── Forged documents detected before UIN issuance         │
│  ├── Data entered matches what documents actually say      │
│  └── Audit trail: which documents proved which identity    │
│                                                              │
│  Document verification is the IAL3 proof in practice.       │
│  (Chapter 2: National ID systems operate at IAL3)           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 19.2 Document Categories

Documents accepted during NBIS enrollment fall into three categories:

```
┌──────────────────────────────────────────────────────────────┐
│              Document Categories                             │
├────────────────────┬─────────────────────────────────────────┤
│ Category           │ Purpose                                │
├────────────────────┼─────────────────────────────────────────┤
│ PRIMARY            │ Proves legal existence and core        │
│ (identity proof)   │ identity attributes (name, DOB).       │
│                    │ One required — cannot be substituted.  │
├────────────────────┼─────────────────────────────────────────┤
│ SECONDARY          │ Supports or corroborates primary.      │
│ (supporting proof) │ Strengthens confidence in identity.    │
│                    │ May substitute primary in some cases.  │
├────────────────────┼─────────────────────────────────────────┤
│ ADDRESS PROOF      │ Proves current or permanent address.   │
│                    │ Required for address fields in record. │
└────────────────────┴─────────────────────────────────────────┘
```

---

## 19.3 Accepted Document Types

```
┌──────────────────────────────────────────────────────────────┐
│              NBIS Accepted Document Reference                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PRIMARY DOCUMENTS:                                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Document          │ Proves          │ Issued by       │  │
│  ├───────────────────┼─────────────────┼─────────────────┤  │
│  │ Birth certificate │ Name, DOB,      │ Civil Registry  │  │
│  │                   │ parents         │                 │  │
│  ├───────────────────┼─────────────────┼─────────────────┤  │
│  │ Passport          │ Name, DOB,      │ Government      │  │
│  │                   │ nationality,    │ (Foreign Aff.)  │  │
│  │                   │ photo           │                 │  │
│  ├───────────────────┼─────────────────┼─────────────────┤  │
│  │ Existing national │ Name, DOB,      │ NID Authority   │  │
│  │ ID card           │ nationality     │ (renewal only)  │  │
│  ├───────────────────┼─────────────────┼─────────────────┤  │
│  │ Residency permit  │ Name, DOB,      │ Immigration     │  │
│  │ (for residents)   │ nationality     │ Authority       │  │
│  └───────────────────┴─────────────────┴─────────────────┘  │
│                                                              │
│  SECONDARY DOCUMENTS (supporting):                           │
│  ├── School certificate / transcript                        │
│  ├── Marriage certificate (name change evidence)            │
│  ├── Military service record                               │
│  ├── Hospital birth record                                 │
│  ├── Community elder affidavit (rural areas)               │
│  └── Court order (name/status change)                      │
│                                                              │
│  ADDRESS PROOF DOCUMENTS:                                    │
│  ├── Utility bill (electricity, water — last 3 months)     │
│  ├── Bank statement (last 3 months)                        │
│  ├── Tenancy agreement (lease contract)                    │
│  ├── Official letter from employer                         │
│  └── Letter from municipality                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 19.4 Document Verification Layers

Document verification happens at multiple layers — physical, digital, and data:

```
┌──────────────────────────────────────────────────────────────┐
│              Document Verification Layers                    │
└──────────────────────────────────────────────────────────────┘

LAYER 1: PHYSICAL INSPECTION (officer — at center)
  Officer examines original document:
  ├── Is the document genuine (not photocopied)?
  ├── Are security features present?
  │   ├── Watermark (visible under light)
  │   ├── Hologram (shifts color at angle)
  │   ├── Embossed seal (raised impression)
  │   ├── Microprinting (visible under magnifier)
  │   └── Security thread (embedded in paper)
  ├── UV light check:
  │   ├── Fluorescent fibers visible under UV
  │   ├── UV-reactive security patterns appear
  │   └── Genuine government paper glows correctly
  ├── Is the document within validity period?
  ├── Are there signs of tampering?
  │   ├── Altered text (different ink, correction fluid)
  │   ├── Lamination peeling (photo substitution)
  │   └── Page replacement (different paper type)
  └── Does the photo match the person present?

LAYER 2: DOCUMENT SCAN + DIGITIZATION
  Officer scans document on flatbed scanner:
  ├── Minimum resolution: 600 DPI
  ├── Both sides scanned (front + back)
  ├── Scan stored in enrollment packet (S3, encrypted)
  └── Scan available for future audit review

LAYER 3: OCR EXTRACTION
  System extracts data from scan:
  ├── Name (first, middle, last)
  ├── Date of birth
  ├── Gender
  ├── Document number
  ├── Issue date + expiry date
  ├── Issuing authority
  └── MRZ zone (if passport / ID card)

LAYER 4: DATA CROSS-CHECK
  OCR result vs officer-entered demographics:
  ├── Name comparison (normalized match)
  ├── DOB comparison (exact match)
  ├── Gender comparison (exact match)
  ├── Nationality comparison (exact match)
  └── Discrepancy? → Officer alerted for manual review

LAYER 5: DOCUMENT AUTHENTICITY CHECK (automated)
  System checks document against known patterns:
  ├── Document number format validation
  │   (BH passport: starts with specific prefix/pattern)
  ├── Check digit verification (ICAO MRZ checksum)
  ├── Issue date vs expiry date logic check
  │   (expiry must be after issue)
  └── Issuing authority code validation

LAYER 6: CIVIL REGISTRY CROSS-CHECK (optional)
  If civil registry API is available:
  ├── Query civil registry with document number
  ├── Confirm person with this name/DOB exists
  ├── Confirm document was genuinely issued
  └── Flag: record exists but document is reported stolen?
```

---

## 19.5 MRZ — Machine Readable Zone

Modern passports and some national ID cards include a **Machine Readable Zone (MRZ)** — two or three lines of standardized text at the bottom of the biographical page that can be scanned and parsed automatically.

```
┌──────────────────────────────────────────────────────────────┐
│              MRZ Structure and Parsing                       │
└──────────────────────────────────────────────────────────────┘

PASSPORT MRZ (TD3 format — 2 lines × 44 characters):

Line 1:
P<BHNAHMAD<<HAMZA<AHMED<<<<<<<<<<<<<<<<<<<<<
│ │ │   │  │
│ │ │   │  └── Name (filler: <)
│ │ │   └───── Issuing country (BHN = Bahrain)
│ │ └───────── Document type (P = Passport)
│ └─────────── Subtype (<)
└───────────── Type indicator (P)

Line 2:
BH1234567<8BHN9005153M3101314<<<<<<<<<<<<<<2
│        │ │  │       │ │      │
│        │ │  │       │ │      └── Check digit
│        │ │  │       │ └──────── Personal number
│        │ │  │       └────────── Sex (M/F)
│        │ │  └────────────────── Expiry date (YYMMDD)
│        │ └───────────────────── Nationality
│        └───────────────────────  DOB (YYMMDD) + check
└──────────────────────────────── Document number + check

MRZ CHECK DIGIT ALGORITHM:
  Weights: 7 3 1 7 3 1 7 3 1 ...
  Values:  A=10 B=11 ... Z=35, 0-9=face value, <=0

  Example: document number "BH1234567"
  B=11, H=17, 1=1, 2=2, 3=3, 4=4, 5=5, 6=6, 7=7
  × 7,  3,    1,   7,   3,   1,   7,   3,   1
  =77 + 51 + 1 + 14 + 9 + 4 + 35 + 18 + 7 = 216
  216 mod 10 = 6  ← check digit must be 6

PARSING IN CODE (Java example):
  MrzParser parser = new MrzParser();
  MrzRecord record = parser.parse(mrzLine1, mrzLine2);

  String name        = record.getSurname() + " " + record.getGivenNames();
  LocalDate dob      = record.getDateOfBirth();
  String nationality = record.getNationality();
  boolean valid      = record.isValid();  // all check digits pass

  if (!valid) {
    throw new DocumentException("MRZ check digit failure — document may be tampered");
  }
```

---

## 19.6 Document Fraud Patterns

Enrollment officers must be trained to detect common fraud patterns:

```
┌──────────────────────────────────────────────────────────────┐
│              Common Document Fraud Patterns                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. PHOTO SUBSTITUTION                                       │
│  Original photo removed, fraudster's photo inserted.        │
│  Detection:                                                  │
│  ├── Lamination disturbance around photo area              │
│  ├── Different paper texture behind photo                   │
│  ├── UV check: security pattern broken at photo border     │
│  └── Facial match: does photo match person present?        │
│                                                              │
│  2. DATA ALTERATION                                          │
│  Name, DOB, or document number changed chemically.          │
│  Detection:                                                  │
│  ├── Ink inconsistency (different color/sheen)             │
│  ├── Chemical erasure marks under UV light                 │
│  ├── MRZ check digit failure (number changed but not MRZ)  │
│  └── Indentations from original writing (oblique light)    │
│                                                              │
│  3. COUNTERFEIT DOCUMENT                                     │
│  Entirely fabricated document (not a genuine one altered).  │
│  Detection:                                                  │
│  ├── Security features absent or incorrectly reproduced    │
│  ├── Paper weight / texture wrong                          │
│  ├── Document number does not match issuing authority      │
│  │   format or sequence                                    │
│  ├── Civil registry cross-check: number not found          │
│  └── MRZ format non-compliant with ICAO standard          │
│                                                              │
│  4. GENUINE DOCUMENT — WRONG PERSON                          │
│  Real document, real person, wrong claimant.                │
│  (Sibling using sibling's passport)                         │
│  Detection:                                                  │
│  ├── Photo does not match person present                   │
│  ├── Physical appearance vs stated age inconsistency       │
│  └── Biometric deduplication: fingerprints match someone   │
│      already enrolled under different name                  │
│                                                              │
│  5. SYNTHETIC IDENTITY                                       │
│  Documents for a person who never existed.                  │
│  Birth certificate fabricated for a fictional person.       │
│  Detection:                                                  │
│  ├── Civil registry: birth record not found                │
│  ├── Hospital birth records do not corroborate             │
│  └── Biometrics not in any database (no prior presence)    │
│      BUT genuinely new person also has no prior records    │
│      → Hardest fraud to detect at enrollment               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 19.7 OCR Engine Design

The OCR (Optical Character Recognition) engine extracts text from scanned documents:

```
┌──────────────────────────────────────────────────────────────┐
│              OCR Pipeline for NBIS                           │
└──────────────────────────────────────────────────────────────┘

INPUT: Scanned document image (600 DPI, TIFF or JPEG2000)
         │
         ▼
STEP 1: Image Pre-processing
  ├── Deskew (straighten tilted scan)
  ├── Denoise (remove scanner artifacts)
  ├── Binarize (convert to black/white)
  ├── Contrast enhancement
  └── Resolution normalization (to 300 DPI for OCR)

STEP 2: Document Type Detection
  ├── Template matching against known document templates
  │   (Bahrain passport template, CPR template, etc.)
  ├── Identify field zones on the document
  └── Detect MRZ zone (bottom of page)

STEP 3: Zone OCR
  ├── Name zone → extract name text
  ├── DOB zone → extract date
  ├── Gender zone → extract M/F
  ├── Document number zone → extract alphanumeric
  └── MRZ zone → parse according to ICAO TD1/TD2/TD3

STEP 4: Post-processing
  ├── Name: apply name normalization rules
  ├── DOB: parse various formats (15/05/1990, 1990-05-15)
  ├── Gender: normalize to M/F
  └── Document number: apply format regex validation

STEP 5: Confidence Scoring
  Each extracted field has a confidence score (0–100):
  ├── Confidence > 85: accept automatically
  ├── Confidence 60–85: flag for officer review
  └── Confidence < 60: reject OCR, officer enters manually

STEP 6: Cross-Validation
  OCR result vs officer-entered data:
  ├── Name: normalized comparison
  │   Match score ≥ 80%: green
  │   Match score 60–79%: yellow (officer confirms)
  │   Match score < 60%: red (officer must re-check)
  └── DOB: exact match required (or flag)

OUTPUT:
  {
    "documentType":   "PASSPORT",
    "documentNumber": "BH1234567",
    "extractedName":  "Hamza Ahmed Rafique",
    "extractedDob":   "1990-05-15",
    "extractedGender":"M",
    "mrzValid":       true,
    "overallConfidence": 94.3,
    "fieldScores": {
      "name": 97.1,
      "dob":  99.8,
      "gender":99.9,
      "docNum":92.4
    }
  }
```

---

## 19.8 Document Verification for Special Cases

### 19.8.1 Late Registration (No Birth Certificate)

```
┌──────────────────────────────────────────────────────────────┐
│              Late Registration — No Birth Certificate        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem:                                                    │
│  Adult was never registered at birth.                       │
│  Has no birth certificate. No primary document.             │
│  Common in rural areas or post-conflict regions.            │
│                                                              │
│  LATE REGISTRATION PROCESS:                                  │
│                                                              │
│  Stage 1: Evidence gathering (multiple supporting docs)      │
│  ├── School records (name, approximate DOB)                 │
│  ├── Hospital birth record (if available)                   │
│  ├── Religious ceremony record (naming ceremony)            │
│  ├── Community elder affidavit (2 elders, notarized)       │
│  └── Employer records (name, approximate age)              │
│                                                              │
│  Stage 2: Civil registry late registration                   │
│  ├── Civil Registry issues late birth certificate          │
│  ├── Marked as: LATE_REGISTERED                            │
│  └── May have APPROXIMATE DOB flag                         │
│                                                              │
│  Stage 3: NBIS enrollment with late birth certificate        │
│  ├── Process same as normal enrollment                     │
│  ├── Document type: LATE_BIRTH_CERTIFICATE                  │
│  └── Flag in record: LATE_REGISTRATION = true              │
│                                                              │
│  Document combination required (minimum):                    │
│  2 supporting documents + civil registry approval           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 19.8.2 Newborn Enrollment

```
┌──────────────────────────────────────────────────────────────┐
│              Newborn Enrollment                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Documents required:                                         │
│  ├── Hospital birth notification (immediate)               │
│  ├── Birth certificate (issued within days)                │
│  └── Parent's national ID cards (both parents)             │
│                                                              │
│  Process:                                                    │
│  ├── Parents enroll newborn at hospital or center          │
│  ├── No biometrics for age < 5 (too small/unstable)        │
│  ├── Face photo captured (newborn face)                    │
│  ├── Parent UINs linked to child record                    │
│  └── Child UIN issued (no physical card yet, digital VC)  │
│                                                              │
│  Re-enrollment at age 5:                                    │
│  ├── Parents bring child to center                         │
│  ├── Fingerprints captured (stable from age 5)            │
│  ├── Iris captured                                          │
│  ├── Face updated                                           │
│  └── Same UIN retained — record updated                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 19.8.3 Foreign National / Resident

```
┌──────────────────────────────────────────────────────────────┐
│              Foreign National Enrollment                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Documents required:                                         │
│  ├── Foreign passport (primary — name, DOB, nationality)   │
│  ├── Visa / residency permit (right to be in country)      │
│  ├── Employment contract (if work visa)                    │
│  └── Tenancy agreement (address proof)                     │
│                                                              │
│  Special verification steps:                                 │
│  ├── Passport: ICAO MRZ scan + chip read (if ePassport)   │
│  ├── Visa: check validity + immigration system API         │
│  ├── Cross-check: passport nationality matches visa type   │
│  └── Employment: employer ID registered in country?        │
│                                                              │
│  Record type: RESIDENT (not CITIZEN)                        │
│  UIN linked to: residency permit expiry date                │
│  Automatic alert: 90 days before permit expiry             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 19.9 Document Storage and Retention

```
┌──────────────────────────────────────────────────────────────┐
│              Document Storage Architecture                   │
└──────────────────────────────────────────────────────────────┘

STORAGE:
  Scanned documents → S3 (nbis-document-scans)
  ├── Encryption: SSE-KMS (per-citizen key)
  ├── Access: extremely restricted (audit + legal only)
  ├── Versioning: enabled (track all uploads)
  └── Object Lock: COMPLIANCE mode

  Reference in enrollment packet:
  "documents": [{
    "type": "BIRTH_CERTIFICATE",
    "s3Ref": "s3://nbis-doc-scans/UIN-123/BC-20250115.enc",
    "sha256": "a1b2c3...",    ← hash of original scan
    "capturedAt": "2025-01-15T10:30:00Z"
  }]

RETENTION POLICY:
  ├── Active citizen: retain for lifetime of identity record
  ├── After revocation/death: retain 30 years (legal)
  ├── After retention period: secure deletion
  └── Audit log: which documents were scanned, retained forever

ACCESS CONTROL:
  Who can access document scans:
  ├── System Admins: YES (for specific UIN, with audit)
  ├── Law enforcement: YES (court order required)
  ├── Relying parties: NO (never)
  ├── Citizens: YES (own documents only via portal)
  └── Enrollment officers: NO (after submission)

S3 LIFECYCLE POLICY:
  Active: S3 Standard (instant access)
  After 1 year: S3 Standard-IA (infrequent access, cheaper)
  After 5 years: S3 Glacier Instant Retrieval
  After 30 years post-revocation: S3 Glacier Deep Archive
  → Then: secure deletion scheduled
```

---

## 19.10 Document Verification Workflow in Code

```
┌──────────────────────────────────────────────────────────────┐
│              Document Verification Flow — Code Logic         │
└──────────────────────────────────────────────────────────────┘

// Registration Processor — Document Verification Stage

class DocumentVerificationProcessor {

  VerificationResult verify(EnrollmentPacket packet) {

    List<DocumentScan> docs = packet.getDocuments();

    // Step 1: Check required documents present
    if (!hasPrimaryDocument(docs)) {
      return VerificationResult.fail(
        "DOC_001", "No primary identity document provided"
      );
    }

    // Step 2: Document type-specific validation
    for (DocumentScan doc : docs) {
      switch (doc.getType()) {

        case PASSPORT:
          validatePassport(doc);
          break;

        case BIRTH_CERTIFICATE:
          validateBirthCertificate(doc);
          break;

        case RESIDENCY_PERMIT:
          validateResidencyPermit(doc);
          break;
      }
    }

    // Step 3: OCR cross-check
    OCRResult ocr = ocrEngine.extract(docs.getPrimary());
    DemographicMatchResult match =
      compareOcrToEntered(ocr, packet.getDemographics());

    if (match.getScore() < 60) {
      return VerificationResult.review(
        "DOC_002", "Name or DOB mismatch between document and entry",
        match
      );
    }

    // Step 4: MRZ validation (if passport/ID card)
    if (hasMRZ(docs.getPrimary())) {
      MRZResult mrz = mrzParser.parse(docs.getPrimary().getScan());
      if (!mrz.isValid()) {
        return VerificationResult.fail(
          "DOC_003", "MRZ check digit failure — possible tampering"
        );
      }
    }

    // Step 5: Document expiry check
    if (isExpired(docs.getPrimary())) {
      return VerificationResult.fail(
        "DOC_004", "Primary document is expired"
      );
    }

    // Step 6: Civil registry check (if available)
    if (civilRegistryAvailable) {
      CivilRegistryResult cr =
        civilRegistry.verify(docs.getPrimary());
      if (cr.isReportedLostOrStolen()) {
        return VerificationResult.fail(
          "DOC_005", "Document reported lost or stolen"
        );
      }
    }

    // All checks passed
    return VerificationResult.success();
  }
}
```

---

## 19.11 Document Verification Error Codes

```
┌──────────────────────────────────────────────────────────────┐
│              Document Verification Error Codes               │
├────────────┬─────────────────────────────────────────────────┤
│ Code       │ Meaning                                        │
├────────────┼─────────────────────────────────────────────────┤
│ DOC_001    │ No primary identity document provided           │
│ DOC_002    │ Name / DOB mismatch between document and entry │
│ DOC_003    │ MRZ check digit failure (possible tampering)   │
│ DOC_004    │ Primary document is expired                     │
│ DOC_005    │ Document reported lost or stolen               │
│ DOC_006    │ Document type not accepted for this enrollment │
│ DOC_007    │ Document scan quality too low (re-scan needed) │
│ DOC_008    │ OCR confidence below threshold (manual review) │
│ DOC_009    │ Document number format invalid                 │
│ DOC_010    │ Civil registry record not found               │
│ DOC_011    │ Document issued in future (impossible date)   │
│ DOC_012    │ Supporting document too old (> 3 months)      │
│ DOC_013    │ Guardian consent missing for minor            │
│ DOC_014    │ Document country of issuance not recognized   │
└────────────┴─────────────────────────────────────────────────┘
```

---

## 19.12 Audit Trail for Document Verification

```
┌──────────────────────────────────────────────────────────────┐
│              Document Verification Audit Log Entry           │
└──────────────────────────────────────────────────────────────┘
{
  "eventId":         "EVT-20250115-DOC-001",
  "eventType":       "DOCUMENT_VERIFIED",
  "timestamp":       "2025-01-15T10:35:00Z",
  "packetId":        "PKT-20250115-001",
  "centerId":        "CENTER-001",
  "operatorId":      "OFR-007",
  "documents": [{
    "type":          "PASSPORT",
    "documentNumber":"BH1234567",  ← stored for audit (not PII)
    "issuingCountry":"BH",
    "expiryDate":    "2031-01-14",
    "mrzValid":      true,
    "ocrConfidence": 94.3,
    "verificationResult":"PASSED"
  }],
  "overallResult":   "PASSED",
  "reviewRequired":  false,
  "traceId":         "abc-123-xyz"
}
```

---

## 19.13 Integration with Civil Registry

```
┌──────────────────────────────────────────────────────────────┐
│              Civil Registry API Integration                  │
└──────────────────────────────────────────────────────────────┘

SCENARIO: NBIS calls Civil Registry to verify birth certificate

  NBIS Registration Processor
          │
          │ GET /civil-registry/verify
          │ { "documentType": "BIRTH_CERTIFICATE",
          │   "documentNumber": "BC-2025-001234",
          │   "name": "Hamza Rafique",
          │   "dob": "1990-05-15" }
          │
          ▼ (mTLS — mutual authentication)
  Civil Registry API
          │
          ├── Document found? YES
          ├── Name matches? YES
          ├── DOB matches? YES
          ├── Status: VALID (not cancelled/superseded)
          └── Lost/stolen: NO
          │
          ▼
  Response:
  {
    "verified": true,
    "status":   "VALID",
    "confidence":"HIGH"
  }

INTEGRATION PATTERNS:
  ├── Real-time API call during enrollment processing
  ├── Batch verification (nightly check of pending records)
  └── Event-driven: civil registry sends events to NBIS
      when documents are cancelled or superseded

WHEN CIVIL REGISTRY API IS DOWN:
  ├── Do not block enrollment (soft dependency)
  ├── Flag record: CIVIL_REGISTRY_NOT_VERIFIED
  ├── Queue for retry when registry comes back online
  └── Supervisor can manually clear after physical review
```

---

## 19.14 AWS Architecture for Document Verification

```
┌──────────────────────────────────────────────────────────────┐
│          Document Verification AWS Architecture              │
└──────────────────────────────────────────────────────────────┘

Registration Center
  │ Scan uploaded with packet
  ▼
S3 (nbis-enrollment-packets)
  │ SSE-KMS encrypted
  │
  ▼ SQS triggers
Registration Processor (ECS Fargate)
  │
  ├──► S3 GetObject (retrieve scan)
  │
  ├──► AWS Textract (OCR service)
  │    AnalyzeDocument API:
  │    ├── Extract key-value pairs
  │    ├── Extract tables
  │    └── Extract MRZ text
  │
  ├──► Amazon Rekognition (face comparison)
  │    CompareFaces API:
  │    ├── Face in document photo vs live photo
  │    └── Similarity score ≥ 90% → match
  │
  ├──► Civil Registry API (via VPC Link)
  │    mTLS authenticated call
  │
  ├──► DynamoDB (update enrollment status)
  │    DOCUMENT_VERIFIED or DOCUMENT_REVIEW
  │
  └──► CloudWatch Logs (audit entry)

AWS SERVICES USED:
  ├── Amazon Textract  → OCR (replaces custom OCR engine)
  ├── Amazon Rekognition → Face match doc photo vs live photo
  ├── S3 + KMS         → Document scan storage + encryption
  ├── ECS Fargate      → Verification processor
  ├── DynamoDB         → Status tracking
  └── CloudWatch       → Audit logging
```

---

## 19.15 MOSIP Reference

```
┌──────────────────────────────────────────────────────────────┐
│              MOSIP Document Verification Reference           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MOSIP document verification happens in two stages:         │
│                                                              │
│  STAGE 1: At Registration Client (client-side)              │
│  ├── Officer selects document type from dropdown            │
│  ├── Officer scans document (flatbed scanner SDK)          │
│  ├── Document stored in local encrypted DB (offline)        │
│  └── Officer manually verifies physical security features  │
│                                                              │
│  STAGE 2: In Registration Processor (server-side)           │
│  ├── Document Validator stage in Kafka pipeline            │
│  ├── Pluggable: country can connect custom OCR engine      │
│  ├── MRZ parsing: built-in MOSIP MRZ parser                │
│  └── Civil registry: pluggable external API integration    │
│                                                              │
│  MOSIP document schema (configurable per country):          │
│  {                                                           │
│    "docTypeCodes": [                                         │
│      { "code": "POI", "name": "Proof of Identity",         │
│        "documents": ["PASSPORT", "BIRTH_CERTIFICATE"] },   │
│      { "code": "POA", "name": "Proof of Address",         │
│        "documents": ["UTILITY_BILL", "BANK_STATEMENT"] }   │
│    ]                                                         │
│  }                                                           │
│                                                              │
│  Amazon Textract vs MOSIP's default OCR:                    │
│  ├── MOSIP default: Tika/Tesseract (open source)           │
│  ├── AWS deployment: replace with Amazon Textract           │
│  └── Textract significantly better for:                    │
│      multi-language docs, handwritten fields, low-quality  │
│      scans common in developing country deployments        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 19.16 Key Terms

| Term | Definition |
|------|-----------|
| **Document verification** | Confirming presented documents are genuine, valid, and sufficient |
| **Primary document** | Core identity proof — birth certificate, passport, national ID |
| **Secondary document** | Supporting proof — school records, employer letter, affidavit |
| **MRZ** | Machine Readable Zone — two lines of standardized text on passports |
| **TD1/TD2/TD3** | ICAO MRZ format types — TD3 is passport (2×44 chars) |
| **Check digit** | Calculated digit that validates an MRZ field has not been altered |
| **OCR** | Optical Character Recognition — extracting text from scanned images |
| **Amazon Textract** | AWS OCR service — extracts text, key-value pairs from documents |
| **Amazon Rekognition** | AWS AI service — face comparison, face detection |
| **Late registration** | Enrolling a person who was never registered at birth |
| **Photo substitution** | Fraud where original passport photo is replaced with fraudster's photo |
| **Synthetic identity** | Fraudulent identity using fabricated documents for a non-existent person |
| **ICAO** | International Civil Aviation Organization — sets passport standards |
| **Civil registry cross-check** | Verifying document against civil registry's authoritative database |
| **OCR confidence score** | How certain the OCR engine is about an extracted value (0–100) |
| **Document scan resolution** | 600 DPI minimum for reliable OCR and security feature detection |
| **S3 Object Lock** | Prevents document scans from being deleted during retention period |

---

## 19.17 Key Takeaways

- **Document verification is the IAL3 gate** — it is what separates a national ID system from a self-asserted identity system. Without it, anyone can claim any identity.
- **Physical inspection cannot be automated away** — UV lights, holograms, embossed seals, and tactile security features require a trained human eye. Officers must be trained, not just given a checklist.
- **MRZ check digits are your tamper detector** — if anyone alters the document number, name, or DOB but forgets to update the MRZ check digits, the validation fails automatically. Never skip MRZ validation.
- **OCR is a tool, not the truth** — OCR confidence scores below 85% mean the officer must manually verify. Low-quality scans, handwritten fields, and worn documents will produce low confidence. Design for this.
- **Civil registry integration is a soft dependency** — the NBIS enrollment must not stop because the civil registry API is temporarily down. Flag the record, queue for retry, let supervisors manually clear.
- **Late registration is a real operational scenario** — especially in first-generation NBIS rollouts where birth registration rates are low. The system must support a multi-document evidence path.
- **Amazon Textract + Rekognition replaces custom OCR** — on AWS, there is no reason to build a custom OCR pipeline. Textract handles multilingual documents, and Rekognition handles face comparison. Use the managed services.
- **Document scans are more sensitive than biometrics** — they contain full document numbers, family names, addresses, and official seals. Restrict access even more tightly than biometric templates. Only law enforcement with court order and citizens viewing their own.

---

## 19.18 What Comes Next

| Chapter | Topic |
|---------|-------|
| Chapter 20 | Fingerprint Capture — device types, NFIQ2, rolled vs slap, exceptions |
| Chapter 21 | Face Capture — ICAO compliance, pose, lighting, liveness detection |
| Chapter 22 | Iris Capture — NIR camera, IrisCode, quality metrics |

---

*Chapter 19 of 116 — Master NBIS Developer Handbook*
*Author: Hamza Rafique*
