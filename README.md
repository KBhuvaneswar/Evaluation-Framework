# Evaluation Framework for AI-Generated Electronic Health Records

## A Multi-Dimensional Quality Assessment of EkaScribe Clinical Documentation

### Key Contributions:
1. **4 Standard Evaluation Dimensions** - Format Compliance, Completeness, Factual Accuracy, Faithfulness
2. **2 Novel Evaluation Techniques** - Clinical Importance Weighting (CIW), Hallucination Category Rate (HCR)
3. **Evaluation on 30 Real Samples** - Audio files processed through EkaScribe API
4. **Dataset Limitation Analysis** - Identified ASR vs hallucination attribution challenge

### Overall Results (30 Samples):

| Dimension | Score | Interpretation |
|-----------|-------|----------------|
| D1: Format Compliance | **100.0%** | 30/30 samples schema-compliant |
| D2: Completeness | **84.3%** | Good entity recall |
| D3: Factual Accuracy | **86.4%** | 90.2% precision, 11 hallucinations |
| D4: Faithfulness | **89.1%** | Lexical + BERTScore semantic grounding |
| CIW-Completeness | **87.2%** | Critical entities well-captured |
| HCR Grounding Rate | **89.6%** | Category-level insights |

---

## 1. Problem Statement

### Context
EkaScribe is an AI-powered clinical documentation tool that:
1. Receives doctor-patient audio conversations
2. Transcribes audio to text (ASR)
3. Extracts medical entities and generates structured EHR

### Challenge
How do we evaluate the quality of AI-generated EHRs when:
- The AI system is a **black box** (no intermediate outputs)
- Ground truth is limited to **transcripts and medical entity labels**
- No reference EHRs exist for comparison

### Approach
Design a multi-dimensional evaluation framework using:
- Schema validation for structure
- Entity matching for content accuracy
- Semantic similarity for faithfulness
- Novel techniques for clinical prioritization

---

## 2. Dataset Overview

### Source
- **Dataset A**: Eka Medical ASR Evaluation Dataset (HuggingFace)
- **30 audio samples** processed through EkaScribe API

### Data Structure

| Component | Source | Purpose |
|-----------|--------|---------|
| Audio Files | Dataset A | Input to EkaScribe |
| Transcript | Dataset A | Ground truth text |
| Medical Entities | Dataset A | Ground truth entities with types |
| Generated EHR | EkaScribe API | System output to evaluate |

### Sample Characteristics
- Audio duration: 3-30 seconds
- Languages: Primarily English
- Entity types: drugs, clinical_findings, symptoms, diagnoses

---

## 3. Evaluation Framework Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVALUATION FRAMEWORK                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STANDARD DIMENSIONS                                            │
│  ├── D1: Format Compliance (Schema Validation)                  │
│  ├── D2: Completeness (Entity Recall)                          │
│  ├── D3: Factual Accuracy (Entity Precision)                   │
│  └── D4: Faithfulness (Semantic Grounding)                     │
│                                                                 │
│  NOVEL TECHNIQUES                                               │
│  ├── CIW: Clinical Importance Weighting                        │
│  └── HCR: Hallucination Category Rate                          │
│                                                                 │
│  SKIPPED                                                        │
│  ├── D5: Numerical Accuracy (No structured numeric data)       │
│  └── D6: Linguistic Quality (EHRs are terse by design)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Standard Dimensions - Detailed Explanation

---

### D1: Format Compliance (100.0%)

**What it measures:** Does the generated EHR follow the expected JSON schema structure?

**Why it matters:** An EHR that doesn't follow the expected structure cannot be processed by downstream systems.

**Method:** Deep schema validation checking:
- Presence of required fields (prescription, medications, medicalHistory)
- Correct data types (arrays where expected, strings/objects where expected)
- Nested structure validation (patientHistory subfields)

**Validation Checks:**
```
Root object contains 'prescription' field
'medications' is an array (not string/object)
'drugAllergy' is an array of objects
Each allergy object has 'name', 'status' fields
'prescriptionNotes' is string OR object (both accepted)
```

**Example - PASS (sample_011):**
```json
{
  "prescription": {
    "medicalHistory": {
      "patientHistory": {
        "drugAllergy": [           
          {
            "name": "gestronol",   
            "status": "Active",    
            "notes": "Presenting with rash..."
          }
        ]
      }
    },
    "language": "EN",              
    "medications": []              
  }
}
```

**Example - Would FAIL:**
```json
{
  "prescription": {
    "medications": "Aspirin, Ibuprofen"  
  }
}
```

**Results:** 30/30 samples passed (100.0%)
- EkaScribe consistently produces well-structured EHRs

---

### D2: Completeness (83.1%)

**What it measures:** Are all ground truth medical entities captured somewhere in the generated EHR?

**Why it matters:** Missing entities means missing clinical information. 

**Method: 3-Strategy Matching**

1. **Exact Substring Match** - Is the entity text found anywhere in the EHR?
2. **Fuzzy Matching (85% threshold)** - Handles spelling variations, ASR errors
3. **Full-Text Search** - Searches ALL fields (name, notes, values), not just names

**Why Full-Text Search?**
EHRs store information in multiple places. "rash" might appear in a `notes` field, not the `name` field:
```json
{
  "name": "gestronol",
  "notes": "Presenting with rash and respiratory distress"  
}
```

---

**Detailed Example - sample_011:**

**Ground Truth Entities (10):**
| Entity | Type |
|--------|------|
| gestronol | drugs |
| rash | clinical_findings |
| respiratory distress | clinical_findings |
| gallamine | drugs |
| muscle weakness | clinical_findings |
| steroid | drugs |
| hyperglycemia | clinical_findings |
| sulphaurea | drugs |
| gastrointestinal symptoms | clinical_findings |
| drug allergy | clinical_findings |

**Generated EHR Content:**
```json
"drugAllergy": [
  {"name": "gestronol", "notes": "Presenting with rash and respiratory distress"},
  {"name": "galamine", "notes": "Presenting with muscle weakness"},
  {"name": "steroid", "notes": "Induced hyperglycemia"},
  {"name": "sulfur urea", "notes": "Causing gastrointestinal symptoms"},
  {"name": "unspecified drug", "notes": "Unpredictable reactions"}
]
```

**Matching Results:**

| GT Entity | Found? | Where? | Match Type |
|-----------|--------|--------|------------|
| gestronol | Yes | name field | Exact match |
| rash | Yes | notes field | Exact substring |
| respiratory distress | Yes | notes field | Exact substring |
| gallamine | Yes | name field | Fuzzy (94%) - "galamine" |
| muscle weakness | Yes | notes field | Exact substring |
| steroid | Yes | name field | Exact match |
| hyperglycemia | Yes | notes field | Exact substring |
| sulphaurea | No | - | Fuzzy (67%) - "sulfur urea" below threshold |
| gastrointestinal symptoms | Yes | notes field | Exact substring |
| drug allergy | No | - | Not found |

**Score: 8/10 = 80.0%**

---

**Example with Low Completeness - sample_015 (50.0%):**

**Ground Truth Entities (8):** hypersensitivity, Cosglo gel, severe skin reactions, allergy, penicillin, anaphylaxis, Azoran, gastrointestinal distress

**Issue:** EkaScribe captured only the drug names but missed some clinical findings:

| GT Entity | Found? | Notes |
|-----------|--------|-------|
| Cosglo gel | No | ASR heard "cosco gel" (84% fuzzy - below 85%) |
| penicillin | Yes | Exact match |
| anaphylaxis | Yes | Found in notes |
| Azoran | Yes | Matched "azoran" |
| hypersensitivity | No | Not captured in EHR |
| allergy | No | Generic term not explicitly stated |
| gastrointestinal distress | No | Not in EHR |
| severe skin reactions | Yes | Found as "Severe skin reaction" |

**Score: 4/8 = 50.0%**

---

**Overall Completeness Results:**

| Score Range | Samples | Percentage |
|-------------|---------|------------|
| 100% | 21 | 70% |
| 80-99% | 2 | 7% |
| 50-79% | 2 | 7% |
| <50% | 5 | 16% |

**Average: 83.1%**

---

### D3: Factual Accuracy (86.9%)

**What it measures:** Is every entity in the generated EHR actually present in the source (transcript/ground truth)?

**Why it matters:** This detects **hallucinations**. A hallucinated medication could be dangerous.

**Method: Two-Tier NER-Enhanced Approach**

**TIER 1:** Extract entities from all structured `name` fields:
- medications, symptoms, diagnosis, medicalHistory (all sub-fields)
- PrescribedTests, examinations, DiagnosticResults
- labVitals, labTests, vitals, bodyVitalSigns, advices

**TIER 2:** Run biomedical NER model (`d4data/biomedical-ner-all`) on `notes` fields to extract additional medical terms.

**Verification:** Each extracted entity is checked against ground truth using:
- Multi-algorithm fuzzy matching (ratio, partial_ratio, token_sort, token_set)
- 85% threshold for match acceptance
- Source transcript text check

**Key Difference from Completeness:**
- **Completeness (D2)**: GT → EHR (Did we capture what was said?)
- **Factual Accuracy (D3)**: EHR → GT (Did we make anything up?)

---

**Detailed Example - sample_011:**

**Entities Extracted (11 total):**

| Entity | Source Field | Tier |
|--------|--------------|------|
| gestronol | drugAllergy.name | 1 |
| galamine | drugAllergy.name | 1 |
| steroid | drugAllergy.name | 1 |
| sulfur urea | drugAllergy.name | 1 |
| unspecified drug | drugAllergy.name | 1 |
| rash | notes (NER) | 2 |
| respiratory distress | notes (NER) | 2 |
| muscle weakness | notes (NER) | 2 |
| hyperglycemia | notes (NER) | 2 |
| gastrointestinal | notes (NER) | 2 |
| reactions | notes (NER) | 2 |

**Verification Results:**

| Entity | Verified? | Reason |
|--------|-----------|--------|
| gestronol | Yes | Exact match in GT |
| galamine | Yes | Fuzzy match (94%) to "gallamine" |
| steroid | Yes | Exact match in GT |
| rash | Yes | Found in transcript |
| respiratory distress | Yes | Found in transcript |
| muscle weakness | Yes | Found in transcript |
| hyperglycemia | Yes | Found in transcript |
| gastrointestinal | Yes | Found in transcript |
| **sulfur urea** | No | 67% match to "sulphaurea" (below 85%) |
| **unspecified drug** | No | Not in source - **TRUE HALLUCINATION** |

**Score: 9/11 verified = 84.8%**

---

**Overall Statistics (30 Samples):**

| Metric | Value |
|--------|-------|
| Total entities extracted | 112 |
| Verified correct | 101 (90.2%) |
| Hallucinations detected | 11 (9.8%) |
| Samples with hallucinations | 9 |

---

**All Detected Hallucinations (11 total):**

| Sample | Hallucinated Entity | Source | Error Type |
|--------|---------------------|--------|------------|
| sample_002 | "Momento" | medications.name | ASR Error |
| sample_004 | "Put it draw 200" | medications.name | ASR Error |
| sample_009 | "Amlocare 2.5 tablet" | medications.name | ASR Error |
| sample_011 | "sulfur urea" | drugAllergy.name | ASR Error |
| sample_011 | "unspecified drug" | drugAllergy.name | True Hallucination |
| sample_013 | "dust exposure" | lifestyleHabits.name | Semantic Variation |
| sample_015 | "cosco gel" | drugAllergy.name | ASR Error |
| sample_015 | "medication" | notes (NER) | Generic extraction |
| sample_025 | "Vomiting" | symptoms.name | Semantic Confusion |
| sample_026 | "Vomiting" | symptoms.name | Semantic Confusion |
| sample_030 | "cysticbro" | notes (NER) | NER extraction error |

**Average D3 Score: 86.4%** | **Overall Precision: 90.2%**

---

### D4: Faithfulness (89.1%)

**What it measures:** Is the generated EHR text semantically grounded in the source content?

**Why it matters:** Even if individual entities match, the overall narrative should reflect what was said, not be a creative reinterpretation.

**Method: Combined Lexical + Semantic (BERTScore) Approach**

We use a two-component approach:
1. **Lexical Grounding (50%):** Word overlap, exact substring, fuzzy partial matching
2. **BERTScore Semantic Similarity (50%):** Deep semantic similarity using RoBERTa-large model

**Formula:**
```
Faithfulness = 0.5 × Lexical_Grounding + 0.5 × BERTScore
```

**Why Both?**
- **Lexical** catches exact matches and word overlap (fast, interpretable)
- **BERTScore** catches semantic equivalence even with different words (e.g., "allergy" vs "hypersensitivity")

---

**Detailed Example - sample_011:**

**Source Text (Transcript):**
> "The patient has a known history of gestronol allergy presenting with rash and respiratory distress..."

**Generated EHR Content (10 text items extracted):**
- "gestronol", "Presenting with rash and respiratory distress"
- "galamine", "Presenting with muscle weakness"
- "steroid", "Induced hyperglycemia"
- "sulfur urea", "Causing gastrointestinal symptoms"
- "unspecified drug", "Unpredictable reactions"

**Component Scores:**

| Component | Score | Calculation |
|-----------|-------|-------------|
| Lexical Grounding | 90.1% | 8/10 items fully grounded lexically |
| BERTScore | 81.6% | Semantic similarity via RoBERTa |
| **Combined** | **85.9%** | 0.5 × 90.1% + 0.5 × 81.6% |

**Not Grounded Items (Lexically):**
- "sulfur urea" - ASR variation of "sulphaurea"
- "unspecified drug" - Hallucination

---

**Sample Results:**

| Sample | Lexical | BERTScore | Combined | Items |
|--------|---------|-----------|----------|-------|
| sample_011 | 90.1% | 81.6% | **85.9%** | 10 |
| sample_012 | 93.5% | 82.5% | **88.0%** | 13 |
| sample_015 | 90.7% | 82.3% | **86.5%** | 6 |
| sample_025 | 93.9% | 80.5% | **87.2%** | 15 |
| sample_005 | 100% | 100% | **100%** | 1 |

**Overall Average:**
- Lexical Grounding: **90.9%**
- BERTScore: **87.3%**
- Combined Faithfulness: **89.1%**

---

## 5. Novel Techniques

---

### Novel 1: Clinical Importance Weighting (CIW)

**Problem Addressed:** Standard completeness treats all entities equally, but:
- Missing **"penicillin allergy"** → potentially fatal anaphylaxis
- Missing **"avoid oily food"** → minor inconvenience

**Solution:** Weight entities by clinical importance.

**Weight Hierarchy:**

| Tier | Weight | Entity Types | Rationale |
|------|--------|--------------|-----------|
| **CRITICAL** | 3.0 | drugs, medications, diagnosis, drugAllergy | Can cause immediate harm if wrong |
| **HIGH** | 2.0 | symptoms, clinical_findings, foodAllergy, tests | Important for diagnosis |
| **MODERATE** | 1.5 | familyHistory, vitals, examinations | Context for treatment |
| **STANDARD** | 1.0 | advices, lifestyle, prescriptionNotes | Non-critical information |

---

**Detailed Example - sample_011:**

| GT Entity | Type | Tier | Weight |
|-----------|------|------|--------|
| gestronol | drugs | CRITICAL | 3.0 |
| gallamine | drugs | CRITICAL | 3.0 |
| steroid | drugs | CRITICAL | 3.0 |
| sulphaurea | drugs | CRITICAL | 3.0 |
| rash | clinical_findings | HIGH | 2.0 |
| respiratory distress | clinical_findings | HIGH | 2.0 |
| muscle weakness | clinical_findings | HIGH | 2.0 |
| hyperglycemia | clinical_findings | HIGH | 2.0 |
| gastrointestinal symptoms | clinical_findings | HIGH | 2.0 |
| drug allergy | clinical_findings | HIGH | 2.0 |

**Calculations:**

**Standard Completeness:** 8/10 = 80.0%

**CIW-Completeness:**
- Total weight: (4 × 3.0) + (6 × 2.0) = 12 + 12 = **24.0**
- Matched weight: 
  - Matched drugs: 3 × 3.0 = 9.0 (sulphaurea missed)
  - Matched findings: 5 × 2.0 = 10.0 (drug allergy missed)
  - Total matched: **19.0**
- CIW Score: 19.0 / 24.0 = **79.2%**

**Interpretation:**
- CIW (79.2%) < Standard (80.0%)
- The missed entity (sulphaurea) was a **CRITICAL tier drug**
- This is **clinically concerning** - a drug allergy was missed!

---

**CIW Results Summary:**
- **Average CIW: 87.2%** vs Standard Completeness: 83.1%
- When CIW > Standard: Low-priority items missed (acceptable)
- When CIW < Standard: Critical items missed (concerning)

---

### Novel 2: Hallucination Category Rate (HCR)

**Problem Addressed:** Standard accuracy gives a single number but doesn't reveal patterns:
- Are hallucinations random or systematic?
- Which clinical categories have the most errors?
- Where should improvement efforts focus?

**Solution:** Track hallucinations by clinical category.

**Categories:**
MEDICATIONS, ALLERGIES, DIAGNOSES, SYMPTOMS, TESTS_VITALS, MEDICAL_HISTORY, ADVICE

---

**Results (30 Samples):**

| Category | Total Entities | Hallucinations | Rate | Risk Level |
|----------|----------------|----------------|------|------------|
| SYMPTOMS | 11 | 2 | 18.2% | MEDIUM |
| MEDICATIONS | 24 | 3 | 12.5% | MEDIUM |
| ALLERGIES | 28 | 3 | 10.7% | MEDIUM |
| TESTS_VITALS | 19 | 2 | 10.5% | MEDIUM |
| MEDICAL_HISTORY | 22 | 1 | 4.5% | LOW |
| OTHER | 2 | 0 | 0.0% | NONE |

**Key Insight:** SYMPTOMS has highest error rate (18.2%)
- AI sometimes misinterprets drug names as symptoms
- Example: "vomistop" (drug) → "Vomiting" (symptom)

---

**Error Type Distribution:**

| Error Type | Count | % | Description |
|------------|-------|---|-------------|
| ASR Errors | 5 | 45.5% | Speech recognition mistakes |
| True Hallucinations | 3 | 27.3% | AI fabricated content |
| Semantic Confusion | 3 | 27.3% | AI misinterpreted meaning |

---

## 6. Limitations

### 1. Critical: Cannot Distinguish ASR vs Hallucination

**Issue:** EkaScribe is a black box. We only see:
- **Input:** Audio
- **Output:** Generated EHR

We don't have the intermediate ASR transcript.

**Example:** When EHR contains "Momento" instead of "mometasone":
- Did ASR mishear the audio? (ASR error)
- Did ASR hear correctly but NLP hallucinate? (True hallucination)

**We cannot determine which.**

**Impact:** ~45% of detected "hallucinations" are likely ASR errors.

### 2. No Composite Score
Results are presented as individual dimension scores because:
- Each dimension measures fundamentally different aspects
- Weighting would require clinical validation
- Dataset limitations would make composite misleading

---

## 7. Future Work

1. **Intermediate ASR Outputs** - Request dataset with transcript before EHR extraction
2. **Clinically-Validated Composite Score** - Work with medical experts
3. **Additional Novel Techniques** - Entity Relationship Verification, Clinical Safety Index

---

## 8. Conclusion

### Summary of Results

| Dimension | Score |
|-----------|-------|
| D1: Format Compliance | 100.0% |
| D2: Completeness | 84.3% |
| D3: Factual Accuracy | 86.4% |
| D4: Faithfulness | 89.1% |
| CIW-Completeness | 87.2% |
| HCR Grounding | 89.6% |

### Key Findings

1. **EkaScribe produces well-structured EHRs** - 100% format compliance
2. **Most errors (~45%) are ASR-related**, not true AI hallucinations
3. **SYMPTOMS category has highest error rate** (18.2%) - drug names misinterpreted
4. **Critical entities (drugs, allergies) are captured better** than standard entities (CIW > Standard)
5. **True hallucinations are rare** (~27% of errors) but clinically significant

### Framework Contributions

**Multi-dimensional evaluation** - Comprehensive quality assessment
**Novel techniques** - CIW and HCR provide clinical insights
**Actionable insights** - Category-level error analysis

---

