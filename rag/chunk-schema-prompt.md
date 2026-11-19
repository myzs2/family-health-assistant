# Chunking Prompt (give this to an LLM together with a source file)

```
# TASK: Split a medical source file into RAG chunks (JSON)

## Goal
I am building a medical RAG system for a family health assistant app.
I will give you a source file (clinical guideline, drug label, lab guide,
textbook chapter, etc.). Split it into chunks and output each chunk as JSON
following the schema below.

Users ask natural questions (Arabic dialect or English), for example
"how is hepatitis B treated?" or "what is the normal HbA1c?". The app finds
chunks using SEQUENTIAL METADATA FILTERS, so every chunk needs consistent
metadata chosen from CLOSED LISTS:

1. condition_group     -> which disease / medical area the chunk is about
2. topic               -> which part of the subject (screening, treatment...)
3. semantic_role       -> what the reader wants (treatment, diagnosis...)
4. patient_population  -> who it applies to (adults, pregnancy...)
5. hybrid search (vector + BM25) on whatever remains

Never invent values outside the closed lists. Consistency matters more than
cleverness: the same kind of content must always get the same labels.

## Chunk schema
{
  "chunk_id": "<document_id>_<zero-padded sequential index>",
  "document_id": "<short id of the source file>",
  "source": {
    "name": "<source name>",
    "version": "<version or year>",
    "type": "guideline | drug_label | diagnostic | textbook | table_figure | other"
  },

  "condition_group": "<exactly ONE value from the condition_group list>",
  "topic": "<exactly ONE value from the topic list>",
  "semantic_role": ["<primary role>", "<optional secondary role>"],
  "patient_population": ["<one or more values from the list>"],

  "section_path": ["<literal heading titles from the source, top to bottom>"],
  "chunk_title": "<short descriptive title>",
  "structural_type": "recommendation | paragraph | table | figure | list | definition | warning",

  "text": "<EXACT original text from the source, verbatim>",

  "entities": {
    "conditions": [],
    "drugs": [],
    "tests": []
  },

  "extensions": { },

  "pdf": {"page": 0, "bbox": [0, 0, 0, 0]},
  "quality_status": "auto | needs_review"
}

## Field rules
- chunk_id: "<document_id>_<zero-padded sequential index>", e.g.
  "who_hbvhcv_0012". The index counts chunks in reading order across the whole
  document. Do NOT use the source's section numbers in the ID.
- text: verbatim from the source. Never paraphrase, summarize, translate, or
  fix wording. It is used for citations and for the final answer.
- section_path: literal heading titles only. If a heading starts with a
  numbering prefix (like "15.7.1" or "Chapter 4"), keep the title and drop the
  number. It is for display only, not for filtering.
- chunk_title: a short English title you write (max 12 words).
- semantic_role: list, first item is primary, at most 2 items.
- entities: only what is explicitly mentioned in the chunk text. Use the
  standard name, plus the abbreviation if the source uses one
  (example: "hepatitis B virus (HBV)").
- extensions: only fields that actually appear in the source (see below).
  If nothing applies, use {}.
- One chunk = one idea. Do not mix different roles or different diseases in
  one chunk unless they cannot be separated.
- Tables: each row or small group of rows is its own chunk. Repeat the column
  headers and units inside the chunk text so a number never loses its meaning.
- Never use numbers, doses, or thresholds from memory. Only from the file.
- If a value does not fit any closed list, use "other" and set
  quality_status to "needs_review".

## MULTIPLE DISEASES IN ONE FILE (important)
Some files cover several diseases (for example a guideline on hepatitis B and
hepatitis C, or a textbook chapter on diabetes and hypertension).
- Decide condition_group PER CHUNK, based on what that specific chunk is about,
  not on the file title.
- Example: in a WHO guideline on "HBV and HCV testing", a chunk about HBsAg
  testing gets "hepatitis_b"; a chunk about anti-HCV testing gets
  "hepatitis_c".
- If a chunk is about several diseases at once and cannot be split, use
  "multi_condition" and list every disease in entities.conditions.
- If a chunk is general and not tied to any disease (for example lab
  logistics, waste disposal, how health-care levels are organized), use
  "general_health_systems".
- Tables that compare diseases: split by disease (one chunk per row or per
  disease column) when possible. Otherwise use "multi_condition".
- Never assign a disease just because it appears in the file title.

## Closed list: condition_group
(use exactly these values; use "other" if none fits)
diabetes
hypertension
cardiovascular            heart disease, heart failure, lipids, stroke
hepatitis_b
hepatitis_c
liver_disease             cirrhosis, fatty liver, other liver conditions
kidney_disease
respiratory               asthma, COPD, pneumonia
infectious_disease        other infections (TB, HIV, malaria, COVID...)
thyroid_endocrine         thyroid and other hormonal disorders
gastrointestinal
neurological              epilepsy, migraine, dementia
mental_health
maternal_pregnancy        pregnancy-related conditions
child_health
cancer
musculoskeletal
dermatology
hematology                anemia, clotting disorders
pain_analgesia
multi_condition
general_health_systems
other

## Closed list: topic (same list for ALL diseases)
screening_prevention      who to screen, vaccination, prevention
epidemiology_background   prevalence, transmission, natural history, burden
symptoms_presentation     signs and symptoms, clinical picture
diagnosis_criteria        how the disease is diagnosed, criteria, algorithms
testing_laboratory        tests, assays, sample types, lab values, reference ranges
staging_severity          severity, stage, risk scores, red flags
treatment_nonpharm        lifestyle, diet, exercise, procedures, surgery
treatment_pharm           medicines, regimens, doses, drug safety
monitoring_followup       follow-up, response monitoring, targets
complications             complications and comorbidities
special_populations       pregnancy, children, elderly, kidney/liver impairment
service_delivery          health-system, lab logistics, implementation
other

## Closed list: semantic_role
definition            what the disease/condition is
symptoms              signs and symptoms
causes_risk_factors   causes, risk factors, transmission routes
diagnosis             diagnostic criteria and how it is diagnosed
lab_interpretation    lab values, reference ranges, what a result means
treatment             treatment plans and recommendations
drug_dosing           dose, route, frequency, duration
drug_safety           side effects, contraindications, interactions, warnings
severity_red_flags    danger signs, when to seek urgent care
prevention_lifestyle  prevention, diet, exercise, lifestyle
monitoring_followup   what to monitor and how often
special_populations   pregnancy, children, elderly, kidney/liver impairment
procedure_info        tests or procedures and how they are done

## Closed list: patient_population
adults, pediatric, adolescents, elderly, pregnancy, lactation,
renal_impairment, hepatic_impairment, people_living_with_hiv, general
(use "general" when the source does not restrict the population)

## Source-specific extensions (inside "extensions", only if present in source)
- Clinical guidelines: recommendation_strength, evidence_certainty,
  target_population, recommendation_origin
- Drug information: drug, dose, unit, route, frequency, duration,
  contraindications
- Diagnostic guidelines: test_name, sensitivity, specificity,
  limit_of_detection, sample_type, reference_range
- Tables and figures: table_label (e.g. "Table 4.1"), headers, rows, units,
  notes

## Output format
- Return a JSON array of chunks only. No commentary, no markdown.
- Valid JSON, UTF-8, one object per chunk.
- If unsure about a classification: choose the closest value, set
  quality_status to "needs_review", and continue.
- If the file is long, I will send it section by section. Continue the chunk
  index from the last number I give you so IDs never repeat.
```

## Examples (text fields are placeholders; in real output copy the source verbatim)

### Example 1: paragraph in a multi-disease guideline
```json
{
  "chunk_id": "who_hbvhcv_0021",
  "document_id": "who_hbvhcv",
  "source": {"name": "WHO Guidelines on Hepatitis B and C Testing", "version": "2017", "type": "guideline"},
  "condition_group": "multi_condition",
  "topic": "service_delivery",
  "semantic_role": ["procedure_info"],
  "patient_population": ["general"],
  "section_path": ["Other practical considerations for testing", "Testing at different levels of the health-care system"],
  "chunk_title": "Testing for HBV and HCV at different health-care levels",
  "structural_type": "paragraph",
  "text": "<verbatim text>",
  "entities": {"conditions": ["hepatitis B virus (HBV)", "hepatitis C virus (HCV)"], "drugs": [], "tests": ["serological markers", "virological tests"]},
  "extensions": {},
  "pdf": {"page": 0, "bbox": [0, 0, 0, 0]},
  "quality_status": "auto"
}
```

### Example 2: table row split by disease
```json
{
  "chunk_id": "who_hbvhcv_0008",
  "document_id": "who_hbvhcv",
  "source": {"name": "WHO Guidelines on Hepatitis B and C Testing", "version": "2017", "type": "guideline"},
  "condition_group": "hepatitis_b",
  "topic": "epidemiology_background",
  "semantic_role": ["causes_risk_factors"],
  "patient_population": ["general"],
  "section_path": ["Background - Epidemiology and natural history"],
  "chunk_title": "Hepatitis B: risk factors and routes of transmission in priority populations",
  "structural_type": "table",
  "text": "<verbatim row, with the column header 'Hepatitis B' repeated>",
  "entities": {"conditions": ["hepatitis B virus (HBV)"], "drugs": [], "tests": []},
  "extensions": {"table_label": "Table 4.1", "headers": ["Key and priority populations", "Hepatitis B"]},
  "pdf": {"page": 0, "bbox": [0, 0, 0, 0]},
  "quality_status": "auto"
}
```

### Example 3: guideline recommendation
```json
{
  "chunk_id": "ada2024_0112",
  "document_id": "ada2024",
  "source": {"name": "ADA Standards of Care", "version": "2024", "type": "guideline"},
  "condition_group": "diabetes",
  "topic": "treatment_pharm",
  "semantic_role": ["treatment", "drug_dosing"],
  "patient_population": ["adults"],
  "section_path": ["Pharmacologic Approaches to Glycemic Treatment", "Initial therapy"],
  "chunk_title": "Initial pharmacologic therapy in type 2 diabetes",
  "structural_type": "recommendation",
  "text": "<verbatim text>",
  "entities": {"conditions": ["type 2 diabetes"], "drugs": ["metformin"], "tests": ["HbA1c"]},
  "extensions": {"recommendation_strength": "<as written in source>", "evidence_certainty": "<as written in source>"},
  "pdf": {"page": 0, "bbox": [0, 0, 0, 0]},
  "quality_status": "auto"
}
```

### Example 4: drug label section
```json
{
  "chunk_id": "paracetamol_label_0005",
  "document_id": "paracetamol_label",
  "source": {"name": "Paracetamol product label", "version": "<year>", "type": "drug_label"},
  "condition_group": "pain_analgesia",
  "topic": "treatment_pharm",
  "semantic_role": ["drug_dosing"],
  "patient_population": ["pediatric"],
  "section_path": ["Dosage and administration", "Pediatric use"],
  "chunk_title": "Paracetamol dosing in children",
  "structural_type": "list",
  "text": "<verbatim text, dose table rows keep their headers and units>",
  "entities": {"conditions": [], "drugs": ["paracetamol"], "tests": []},
  "extensions": {"drug": "paracetamol", "dose": "<as written>", "unit": "<as written>", "route": "<as written>", "frequency": "<as written>"},
  "pdf": {"page": 0, "bbox": [0, 0, 0, 0]},
  "quality_status": "auto"
}
```

### Example 5: lab reference range
```json
{
  "chunk_id": "lab_guide_0040",
  "document_id": "lab_guide",
  "source": {"name": "Clinical laboratory reference guide", "version": "<year>", "type": "diagnostic"},
  "condition_group": "thyroid_endocrine",
  "topic": "testing_laboratory",
  "semantic_role": ["lab_interpretation"],
  "patient_population": ["adults"],
  "section_path": ["Thyroid function tests", "TSH"],
  "chunk_title": "TSH reference range and interpretation",
  "structural_type": "table",
  "text": "<verbatim row with headers and units>",
  "entities": {"conditions": [], "drugs": [], "tests": ["TSH"]},
  "extensions": {"test_name": "TSH", "reference_range": "<as written>", "sample_type": "<as written>"},
  "pdf": {"page": 0, "bbox": [0, 0, 0, 0]},
  "quality_status": "auto"
}
```

## Notes
- Put `pdf.page` and `bbox` with code, not the LLM (LLMs get coordinates wrong).
- Test on the first 20 to 30 chunks of one source and review the labels before
  running on everything.
- Use the same closed lists across every file.
