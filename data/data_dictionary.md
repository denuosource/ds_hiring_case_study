# Data dictionary

Peri-operative extract, six-practice provider network. Study period 2022-01-01 to 2024-12-31
inclusive (36 months). 100,000 patients.

Five data files are shipped, all in `data/`:

| File | Format | Grain |
|---|---|---|
| `surgeries.csv` | CSV, header row | one row per patient / booked procedure |
| `observations.parquet` | Parquet | one row per variable per charting time |
| `medications.csv` | CSV, header row | one row per administration |
| `events.csv` | CSV, header row | one row per recorded event |
| `prior_history.csv` | CSV, header row | one row per prior record |

## General conventions

**Timestamps.** All timestamp columns are UTC. In the CSV files they are written as
`YYYY-MM-DD HH:MM:SS+00:00`; parse them as timezone-aware and they will come back as UTC
without adjustment. In `observations.parquet`, `ts` is stored as a native Parquet timestamp,
nanosecond precision, timezone UTC. No local time is recorded anywhere in the extract and there
is no daylight-saving handling to do.

**Identifiers.** `patient_id` is formatted `P-` plus six digits, `surgery_id` is `S-` plus six
digits. Both are unique across the extract, and within this extract one patient corresponds to
exactly one `surgery_id`. `prior_history.csv` carries `patient_id` only. Joins between the other
four files can be made on either key.

**Encoding and separators.** UTF-8, comma-separated, standard double-quoting. A small number of
`procedure_name` values contain a comma and are quoted in `surgeries.csv`.

**Empty values.** Empty CSV fields are written as zero-length strings rather than a sentinel.
They occur in `surgeries.actual_start_ts` (not every booking has a recorded procedure start),
`events.ts` (empty for the `normal` event type), and the `value` and `unit` columns of
`prior_history.csv` (populated only for the `outpatient_lab` record type). Nulls in
`observations.parquet` are true Parquet nulls in the `value` column, reflecting monitor or
charting gaps.

**Numeric codes.** `procedure_cpt` is a five-character CPT code. It is all digits, so most CSV
readers will infer an integer type; read it as a string if leading-zero safety matters to you
elsewhere.

---

## surgeries.csv

One row per patient: booking, procedure, demographics, comorbidity flags and administrative
fields.

| Column | Type | Description |
|---|---|---|
| `patient_id` | string | Patient identifier. Unique in this file. |
| `surgery_id` | string | Booked procedure identifier. Unique in this file. |
| `practice_id` | string | Practice at which the patient was admitted. `PR-01` to `PR-06`. |
| `admit_ts` | timestamp (UTC) | Moment the patient was admitted and the procedure was booked. Admission and booking are a single decision, recorded once. |
| `surgery_scheduled_ts` | timestamp (UTC) | Theatre slot recorded against the booking. Always populated. Reflects the most recent scheduling decision for the encounter rather than the first one. |
| `actual_start_ts` | timestamp (UTC) | Recorded start of the procedure. Empty where the encounter has no recorded procedure start. |
| `procedure_cpt` | string (5 digits) | CPT code for the booked procedure. See the procedure table below. |
| `procedure_name` | string | Procedure description. One-to-one with `procedure_cpt`. |
| `urgency` | string | Booking urgency assigned at the time of booking. `emergency`, `urgent`, `expedited`, `elective_inpatient`. |
| `age` | integer | Age in years at admission. |
| `sex` | string | `F`, `M`. |
| `weight_kg` | float | Weight in kilograms, one decimal. |
| `height_cm` | float | Height in centimetres, one decimal. |
| `bmi` | float | Body mass index, kg/m2, one decimal. Derived from the two columns above. |
| `asa_class` | integer | ASA physical status classification, 1 to 5. |
| `smoking_status` | string | `never`, `former`, `current`. |
| `anticoagulated` | integer 0/1 | On therapeutic anticoagulation at admission. |
| `diabetes` | integer 0/1 | Comorbidity flag. |
| `heart_failure` | integer 0/1 | Comorbidity flag. |
| `ckd` | integer 0/1 | Chronic kidney disease. Definition maintained by the source system. |
| `copd` | integer 0/1 | Comorbidity flag. |
| `cad` | integer 0/1 | Coronary artery disease. |
| `active_cancer` | integer 0/1 | Comorbidity flag. |
| `cirrhosis` | integer 0/1 | Comorbidity flag. |
| `prior_stroke` | integer 0/1 | Comorbidity flag. |
| `immunosuppressed` | integer 0/1 | Comorbidity flag. |
| `surgeon_volume_indicator` | string | Operating surgeon's caseload band for the procedure. `busy`, `occasional`. |
| `discharge_disposition` | string | Where the patient went at the end of the encounter. `home`, `rehab`, `skilled_nursing`, `icu`, `transferred_acute`, `expired`. |

The nine comorbidity columns are `diabetes`, `heart_failure`, `ckd`, `copd`, `cad`,
`active_cancer`, `cirrhosis`, `prior_stroke`, `immunosuppressed`. All are coded 0/1 with no
missing values.

### Procedure codes

| `procedure_cpt` | `procedure_name` |
|---|---|
| 27130 | Total hip arthroplasty |
| 27447 | Total knee arthroplasty |
| 27236 | Open treatment of femoral neck fracture |
| 47562 | Laparoscopic cholecystectomy |
| 44970 | Laparoscopic appendectomy |
| 49505 | Repair inguinal hernia, initial |
| 44140 | Partial colectomy with anastomosis |
| 43644 | Laparoscopic gastric bypass |
| 35301 | Thromboendarterectomy, carotid |
| 44120 | Enterectomy, resection of small intestine |
| 33533 | Coronary artery bypass, single arterial graft |
| 32480 | Removal of lung, single lobe |
| 61510 | Craniotomy for excision of brain tumor |

---

## observations.parquet

Ward vitals and laboratory results for the inpatient wait, in **long format**: one row per
patient per timestamp per variable.

| Column | Type | Description |
|---|---|---|
| `patient_id` | string (dictionary-encoded) | Patient identifier. |
| `surgery_id` | string (dictionary-encoded) | Booked procedure identifier. |
| `ts` | timestamp[ns, UTC] | Time the observation was charted. |
| `variable` | string (dictionary-encoded) | Variable name. See below. |
| `value` | float32 | Measured value, in the unit given in `unit`. May be null. |
| `unit` | string (dictionary-encoded) | Unit of measure. Exactly one unit per variable across the whole file. |

### Variables and units

Vitals, charted every 2 hours:

| `variable` | `unit` | Description |
|---|---|---|
| `heart_rate` | bpm | Heart rate. |
| `sbp` | mmHg | Systolic blood pressure. |
| `dbp` | mmHg | Diastolic blood pressure. |
| `map` | mmHg | Mean arterial pressure. |
| `spo2` | % | Peripheral oxygen saturation. |
| `resp_rate` | breaths/min | Respiratory rate. |
| `temp_c` | C | Temperature, degrees Celsius. Celsius throughout; no Fahrenheit values are recorded. |

Labs, resulted approximately every 6 hours:

| `variable` | `unit` | Description |
|---|---|---|
| `creatinine` | mg/dL | Serum creatinine. |
| `lactate` | mmol/L | Serum lactate. |
| `hemoglobin` | g/dL | Haemoglobin. |
| `wbc` | 10^9/L | White cell count. |
| `platelets` | 10^9/L | Platelet count. |
| `glucose` | mg/dL | Blood glucose. |
| `potassium` | mmol/L | Serum potassium. |
| `crp` | mg/L | C-reactive protein. |
| `troponin` | ng/mL | Troponin. |

The seven vitals share a timestamp within a charting round; the nine labs share a separate
timestamp within a lab round. Cadence is the same at all six practices. Charting for an
encounter begins at `admit_ts`. Charting for an encounter ceases once the patient leaves the
pre-operative ward.

The extract is taken from the monitoring feed as recorded and includes monitor artifacts: a
small number of implausible values on the vitals (for example a heart rate of 0, or an oxygen
saturation above 100%), consistent with sensor or charting error rather than a true reading.

### Working with the long format

The file has one row per patient, per timestamp, per variable, rather than one row per patient
per timestamp with a column for each vital or lab. Turning it into a wide table means pivoting
on `variable`. Vitals and labs are charted on different clocks, so a pivot across both at once
will leave lab columns null on vitals timestamps and vital columns null on lab timestamps;
handling the two groups separately, or aligning them explicitly, is part of the exercise.

---

## medications.csv

Inpatient medication administrations during the encounter. One row per administration. No
column in this file has missing values.

| Column | Type | Description |
|---|---|---|
| `patient_id` | string | Patient identifier. |
| `surgery_id` | string | Booked procedure identifier. |
| `ts` | timestamp (UTC) | Time of administration. |
| `drug_name` | string | Generic drug name, lower case. |
| `drug_class` | string | One-word indication group for the drug. See below. |
| `dose` | integer | Dose administered, in `dose_unit`. Constant per drug in this extract. |
| `dose_unit` | string | `mg`, `units`, `mmol`, `mcg/min`. |
| `route` | string | `PO`, `IV`, `SC`, `IV infusion`. |

### drug_class and drug_name

| `drug_class` | `drug_name` values | Dose unit | Route |
|---|---|---|---|
| `analgesic` | paracetamol, morphine, oxycodone, ketorolac | mg | PO |
| `antibiotic` | cefazolin, piperacillin-tazobactam, metronidazole, vancomycin | mg | IV |
| `antiemetic` | ondansetron, metoclopramide | mg | IV |
| `anticoagulant` | enoxaparin, heparin, apixaban | mg | SC |
| `insulin` | insulin aspart, insulin glargine | units | SC |
| `antihypertensive` | metoprolol, amlodipine, ramipril | mg | PO |
| `diuretic` | furosemide | mg | IV |
| `electrolyte` | potassium chloride, magnesium sulfate | mmol | IV |
| `sedative` | midazolam, propofol | mg | IV |
| `vasopressor` | noradrenaline, vasopressin, adrenaline | mcg/min | IV infusion |

Route and dose unit are fixed per drug class in this extract. Assignment of a drug to a class is
maintained by the source system.

---

## events.csv

Clinical events recorded during the inpatient wait. This is the source for the outcome.

| Column | Type | Description |
|---|---|---|
| `patient_id` | string | Patient identifier. |
| `surgery_id` | string | Booked procedure identifier. |
| `ts` | timestamp (UTC) | Time the event was recorded. **Empty for `normal` rows.** |
| `event_type` | string | Event recorded. See below. |

| `event_type` | Description |
|---|---|
| `emergency_icu_transfer` | Unplanned transfer to critical care. |
| `vasopressor_initiation` | Vasopressor infusion started. |
| `rapid_response_activation` | Rapid response team called to the ward. |
| `intubation` | Emergency intubation on the ward. |
| `cardiac_arrest` | Cardiac arrest. |
| `death` | Death during the encounter. |
| `normal` | No event recorded for the encounter. |

### Structure

Patients whose wait passed without a recorded event carry exactly one row with `event_type` set
to `normal` and an empty `ts`. `normal` never appears alongside another event type for the same
patient.

Patients who did deteriorate carry one or more timestamped rows and no `normal` row. Where there
is more than one row, they are the cascade as it was recorded on the ward, in time order, so a
patient's rows describe one episode rather than several independent ones.

There is no 0/1 outcome column in this file or anywhere else in the extract.

---

## prior_history.csv

Records from the patient's history in the **three months before `admit_ts`**: outpatient
laboratory results, ED attendances, inpatient admissions, prior procedures and prior
deterioration episodes. All rows are strictly earlier than the patient's `admit_ts`. Patients
with no history in the window are simply absent from this file.

| Column | Type | Description |
|---|---|---|
| `patient_id` | string | Patient identifier. This file has no `surgery_id`. |
| `ts` | timestamp (UTC) | Date and time of the prior record. |
| `record_type` | string | Kind of record. See below. |
| `detail` | string | Free-text-style descriptor, lower case. Vocabulary is fixed per `record_type`. |
| `value` | float | Result value. Populated for `outpatient_lab` only; empty otherwise. |
| `unit` | string | Unit for `value`. Populated for `outpatient_lab` only; empty otherwise. |

### record_type and detail vocabularies

| `record_type` | `detail` values |
|---|---|
| `outpatient_lab` | albumin, creatinine, egfr, hba1c, hemoglobin, wbc |
| `ed_visit` | abdominal pain, chest pain, fall, fever, shortness of breath |
| `inpatient_admission` | cellulitis, copd exacerbation, gi bleed, heart failure exacerbation, pneumonia, urinary tract infection |
| `prior_surgery` | arthroscopy, cataract extraction, cholecystectomy, coronary angioplasty, hernia repair |
| `prior_deterioration` | intubation, rapid response activation, unplanned icu transfer |

`prior_deterioration` records an episode of the same kind as the events in `events.csv`, but
occurring before this admission.

### Outpatient lab units

| `detail` | `unit` |
|---|---|
| `creatinine` | mg/dL |
| `egfr` | mL/min/1.73m2 |
| `hba1c` | % |
| `hemoglobin` | g/dL |
| `albumin` | g/dL |
| `wbc` | 10^9/L |

Outpatient results come from community and referral laboratories rather than the inpatient
analysers, and are reported to two decimal places. A small number of values fall outside the
physiologically plausible range for the assay.

---

## Notes on scope

The extract covers the pre-operative inpatient period only. The intra-operative period is not
represented: there is no anaesthetic record, no theatre observations and no waveform data.
Post-operative ward and critical care data are likewise out of scope, with the exception of
`discharge_disposition` in `surgeries.csv`, which is an end-of-encounter administrative field.
