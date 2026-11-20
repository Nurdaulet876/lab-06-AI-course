# Lab 06 — Design Justification & Accountability Specification

## 1. Operational Boundaries: What the System May Do and Must Refuse

The Tulpar Scholarship Screening Harness is strictly scoped as a **decision-support pre-screening tool** for adult scholarship applicants who have explicitly consented to automated evaluation.

### 1.1 Permitted Operations (What the System May Do)
* **Merit-and-Need Screening (`purpose == "scholarship_screening"`):** Evaluate complete, schema-valid applications from applicants aged `18–40` with `consent_to_automated_processing == True` using only job/scholarship-relevant criteria (`program`, `age`, `gpa`, `exam_score`, `household_income_bracket`, `achievements`, `consent_to_automated_processing`).
* **Automated Clear-Case Triage:** Emit automated `shortlist` (`score >= 50`) or `reject` (`score <= 39`) recommendations only when the model output parses cleanly into valid JSON, passes consistency checks (`my_extra_escalation`), and falls outside the human-review uncertainty band.
* **Human-in-the-Loop Escalation:** Route ambiguous or high-risk cases (`outcome == "escalate"`) to human scholarship committee members for final determination.

### 1.2 Mandatory Refusal Boundaries (What the System Must Refuse)
All refusals occur deterministically in code **before** the LLM is invoked (`0` model calls, `score = None`), preventing unauthorized data processing and prompt-level manipulation:
1. **Schema & Domain Violations (`validate_input` $\rightarrow$ `invalid_input`):** Refuses non-numeric/boolean values or out-of-range metrics (`gpa` outside `0.0–4.0`, `exam_score` outside `0–140`, `household_income_bracket` outside `1–5`, `achievements` outside `0–5`, `age` outside `16–40`).
2. **Purpose Limitation (`refuse` $\rightarrow$ `purpose_not_allowed`):** Refuses any request whose `purpose` is not in `ALLOWED_PURPOSES = {"scholarship_screening"}` (e.g., `rank_by_ethnicity`).
3. **Protection of Minors (`refuse` $\rightarrow$ `minor`):** Refuses automated profiling of applicants under age `18` (`age < 18`), requiring verified legal-representative procedures outside the automated pipeline.
4. **Consent Enforcement (`refuse` $\rightarrow$ `no_consent`):** Refuses processing whenever `consent_to_automated_processing` is not strictly `True`.
5. **Free-Text Sanitization & Anti-Injection (`my_extra_refusal` $\rightarrow$ `prompt_injection_detected` / `invalid_free_text`):** Refuses empty/oversized (>100 chars) free-text strings (`name`, `university`, `program`) or strings containing role delimiters, newlines, or control directives (`ignore previous`, `system:`, `output:`, `shortlist`, `reject`, `score <num>`).

---

## 2. Ethical and Legal Grounding of Harness Controls

Each active control in `HARNESS` is mapped in `CONTROL_REGISTER` (`control_register.json`) to a named ethical principle and a primary legal provision:

1. **`validate_input`:**
   * *Ethical Principle:* **Robustness and Safety (Accuracy and Data Integrity)** — *UNESCO Recommendation on the Ethics of Artificial Intelligence (2021), Section III.2, paras. 27–29*.
   * *Legal Instrument:* **Law of the Republic of Kazakhstan "On Personal Data and Their Protection" dated May 21, 2013, No. 94-V, Article 25, Paragraph 2, Subparagraph 3** (duty of the owner/operator to ensure accuracy, reliability, and relevance of personal data).
   * *Application:* Prevented the model from scoring corrupted inputs (`GPA 9.9, exam score -5`), which the bare pipeline silently rejected without flagging data corruption.
2. **`blind` (`("name", "university")`):**
   * *Ethical Principle:* **Fairness and Non-Discrimination (Data Minimization)** — *OECD Recommendation of the Council on Artificial Intelligence (2019), Section 1.2 (Human-centred values and fairness)*.
   * *Legal Instrument:* **Regulation (EU) 2016/679 (GDPR), Article 5(1)(c)** (*data minimisation: personal data must be adequate, relevant, and limited to what is necessary*) — *used as an international benchmark*.
   * *Application:* Eliminated the `+2.5`-point flagship university bias (`sign p = 0.002`) and removed culturally/gender-coded name tokens (`10.0%` baseline flip rate on `gender_coded_name`).
3. **`refuse` (built-in + `my_extra_refusal`):**
   * *Ethical Principle:* **Human Rights, Consent, and Purpose Limitation** — *UNESCO Recommendation on the Ethics of Artificial Intelligence (2021), Section III.2, paras. 32–34 (Privacy and Right to Consent)*.
   * *Legal Instrument:* **Law of the Republic of Kazakhstan No. 94-V (May 21, 2013), Article 7, Paragraph 1 and Article 8, Paragraph 1** (*collection and processing of personal data shall be carried out with the verifiable consent of the subject or his/her legal representative strictly within declared purposes*).
   * *Application:* Blocked `5/6` adversarial/ineligible test requests (`purpose_not_allowed`, `minor`, `no_consent`, `invalid_input`, `prompt_injection_detected`) with `0` model calls and `0/400` false-positive refusals on clean applicants.
4. **`log` (`decisions.jsonl`):**
   * *Ethical Principle:* **Transparency, Explainability, and Auditability** — *EU HLEG Ethics Guidelines for Trustworthy AI (2019), Chapter II, Section 1.4 (Transparency and Traceability)*.
   * *Legal Instrument:* **Regulation (EU) 2024/1689 (EU Artificial Intelligence Act), Article 12(1) and Article 19(1)** (*automatic recording of events/logs over the lifetime of high-risk educational AI systems under Annex III, point 3*) — *used as an international benchmark*.
   * *Application:* Recorded all `406` screening events (`400` measured profiles + `6` Part 4 boundary tests) with cryptographic hashes and reason codes.
5. **`escalate` (`borderline=(40, 49)` + `my_extra_escalation`):**
   * *Ethical Principle:* **Human Agency and Oversight** — *EU HLEG Ethics Guidelines for Trustworthy AI (2019), Chapter II, Section 1.1 (Human Agency and Oversight)*.
   * *Legal Instrument:* **Regulation (EU) 2016/679 (GDPR), Article 22(3)** (*right to obtain human intervention and contest automated decisions*) — *used as an international benchmark*.
   * *Application:* Routed `34/400` (`8.5%`) borderline applications (`score` in `[40, 49]`) plus any unparseable replies or merit-decision contradictions (`high_merit_rejected`, `low_merit_shortlisted`, `insufficient_reason`) to human reviewers.

### 2.1 Reflection on Legal Mapping Difficulty (Part 5 Prediction)
The control hardest to tie to a specific statutory provision is **`validate_input`** (alongside the **prompt-injection regex** inside `my_extra_refusal`). Statutes such as Kazakhstan Law No. 94-V (Article 25) and GDPR (Article 5(1)(d)) mandate high-level outcomes—that personal data be "accurate" and "reliable"—but they do not prescribe software-engineering mechanisms such as type/range assertions (`0.0 <= gpa <= 4.0`) or lexical filtering against LLM instruction injection. Bridging that gap requires translating broad statutory accuracy and security duties into concrete technical invariants.

---

## 3. Accountability Record and Auditability (`decisions.jsonl`)

Every call to `screen()` appends one self-contained JSON record (`406` records total in the submitted `decisions.jsonl`: `343` `shortlist`, `24` `reject`, `34` `escalate`, `5` `refuse`) containing `16` structured keys:

* **Identity & Integrity:** `record_id` (deterministic SHA-256 prefix over record content + log sequence index), `pid` (application identifier), `input_sha256` (SHA-256 digest of the exact rendered prompt shown to the model, or of the full profile dictionary on pre-inference refusals), and UTC `timestamp`.
* **Data Minimization in Logs:** `fields_shown` and `fields_withheld` (`["name", "university"]`) record **field names only, never raw applicant values**. Thus, `decisions.jsonl` acts as an immutable audit trail without duplicating sensitive PII, while `input_sha256` allows an auditor holding the source database to verify byte-for-byte what the model saw.
* **Reproducibility & Decision Provenance:** `model_id` (`Qwen/Qwen2.5-3B-Instruct`), `prompt_version` (`v1`), full `harness` configuration snapshot, `raw_output` (or `[no model call] <reason>`), parsed `decision` and `score`, `controls_fired`, final `outcome`, `reason_code`, and boolean `escalated_to_human`.
* **Post-Hoc Audit Reconstruction:** An auditor inspecting `decisions.jsonl` a year later can determine *why* any application was refused (`purpose_not_allowed`, `minor`, `no_consent`, `invalid_input`, `prompt_injection_detected`), why it was escalated (`borderline`, `unparseable`, `high_merit_rejected`), or which exact model/prompt produced an automated score—without re-running the LLM.

---

## 4. Trade-Off Analysis and Justification

1. **Human Review Workload vs. Error Containment (`BORDERLINE = (40, 49)`):**
   In Part 1, every baseline `shortlist` scored $\ge 45$ (`min = 45`) and every `reject` scored $\le 45$ (`max = 45`), with all `7` decision flips clustered in the `40–49` bin (`64/400 = 16.0%` of baseline profiles). Setting `BORDERLINE = (40, 49)` captured this entire uncertainty zone. Because blinding shifted the score distribution, the actual post-harness escalation rate was **`34/400` (`8.5%`)**—a manageable human review workload that eliminates unreviewed automated rejections in the model's decision boundary while leaving `91.5%` of clear cases automated.
2. **Blinding `university` vs. Loss of Institutional Context:**
   Hiding `university` eliminated the statistically significant `+2.5`-point flagship advantage (`p = 0.002`) and reduced prompt tokens by `7.7%` (`77,922` vs. `84,409`). The accepted trade-off is that legitimate differences in academic rigor across institutions can no longer be weighed by the model; however, because the scholarship rules define merit strictly via GPA, entrance exam score, financial need, and achievements, institutional prestige is an illegitimate proxy that must be excluded.
3. **Prompt Perturbation Sensitivity vs. Structural Consistency:**
   Removing `name` and `university` changed the prompt layout, causing `46/400` (`11.5%`) outcomes and `166/400` (`41.5%`) scores to shift (`mean |delta| = 14.01`). We accepted this shift because `34` of those `46` outcome changes were conservative transitions into `escalate` (`22` `reject -> escalate`, `12` `shortlist -> escalate`) rather than silent rejections, ensuring borderline cases receive human review.
4. **Excluding `counterfactual_check` to Preserve Compute Budget:**
   Enabling `counterfactual_check` doubles inference calls (`800` calls instead of `400`) and latency (`~960s` instead of `480.3s`) to detect sensitivity to attributes that `blind` already removes from the prompt (`0.0%` post-blinding flips). Keeping `counterfactual_check=False` achieved zero protected-attribute divergence at `1.00` call/profile.
5. **Strict Regex Refusal vs. False-Positive Risk on Unusual Names:**
   `my_extra_refusal` blocks control tokens (`system:`, `output:`, `shortlist`, `reject`, newlines) in free-text fields. While this achieved `0/400` false refusals on our dataset, an applicant with an unusual substring (e.g., a program or institution containing the word `"reject"`) could be falsely refused; in production, such refusals should trigger an administrative support ticket rather than permanent exclusion.

---

## 5. AI-Use Declaration

The following AI tools were used in this lab work:
- **ChatGPT** - to translating and changing the instruction type from complex to more plain.
- **Claude** - to get information about searching the principles (which sources should I use, in my case it recommended https://adilet.zan.kz/ and https://eur-lex.europa.eu/) and also in helping with choosing the right article.
- **Gemini** - to explain statistical outputs for better understanding.
