# AI-Powered Examination Seat Generator

A lightweight institutional application that imports examination data from Excel, uses an AI API to interpret administrator instructions, and generates a seating plan using a deterministic allocation engine.

> **Core principle:** AI interprets and structures instructions. Python validates data and rules. The allocation algorithm assigns seats. AI does not independently allocate seats or override hard constraints.

## 1. Project goals

- Reduce manual preparation of examination seating plans.
- Support Excel (`.xlsx`) and CSV input.
- Identify incomplete, inconsistent, or potentially conflicting examination data.
- Let coordinators describe supported seating preferences in natural language.
- Generate reproducible, validated room-wise seating plans.
- Export seating plans and validation reports to Excel and PDF.
- Keep the implementation small enough to pilot in one institution.

## 2. Scope

### In scope for the first version

- Student roster import and column mapping.
- Room and capacity import.
- Examination/session and subject data import.
- Data validation and review of ambiguous records.
- AI-assisted interpretation of seating instructions through a cloud API.
- Coordinator review and approval of the structured rules.
- Deterministic seat allocation.
- Hard-constraint validation and a clear infeasibility report.
- Room-wise seating charts, student lookup, and exports.
- Basic history of generated plans and rule approvals.

### Out of scope for the first version

- AI predicting which individual students might cheat.
- Face recognition, camera surveillance, or biometric processing.
- Autonomous relaxation of institutional rules.
- Training or fine-tuning a model.
- Multi-agent orchestration, RAG, vector databases, or a microservice architecture.
- Direct integrations with an institution's SIS/ERP until the Excel workflow is proven.
- Automatic emailing/SMS notifications.

## 3. Suggested technology stack

| Layer | Suggested technology | Purpose |
|---|---|---|
| UI | Streamlit | Minimal, fast-to-build multi-page interface |
| Application logic | Python | Import, validation, orchestration and exports |
| Excel handling | pandas + openpyxl | Read and write `.xlsx` files |
| AI API | Provider adapter; Gemini API first, Groq as optional backup | Convert plain-language instructions to structured rules |
| Rule validation | Pydantic | Validate AI output against an allow-list schema |
| Allocation | Python algorithm initially; OR-Tools CP-SAT if constraints become complex | Deterministic seat allocation |
| Storage | SQLite for pilot; PostgreSQL for institutional deployment | Store metadata, rules, plan versions and audit events |
| Excel export | pandas + openpyxl | Room plans and validation reports |
| PDF export | ReportLab | Printable seating charts |

Keep the first version as a **modular monolith**: one application, separated into clear modules. Do not introduce a separate frontend and backend unless the pilot needs them.

## 4. High-level architecture

```text
                         EXAM COORDINATOR
                                |
                                v
                  STREAMLIT MULTI-PAGE UI
                                |
              +-----------------+------------------+
              |                 |                  |
              v                 v                  v
        Excel Import      AI Rule Assistant    Plan History
              |                 |
              v                 v
        Data Validator    Cloud API Adapter
              |                 |
              |          Structured JSON
              |                 |
              |          Pydantic Validation
              |                 |
              +---------> Human Approval
                                |
                                v
                       Approved Rule Set
                                |
                                v
                      Allocation Engine
                                |
                                v
                       Hard-Rule Validator
                                |
                   +------------+------------+
                   |                         |
                   v                         v
             Valid Seating Plan       Conflict Report
                   |
                   v
           Excel / PDF / Print View
```

### Module responsibilities

- **UI:** Collects uploads, instructions, confirmations, room selections and export requests.
- **Excel importer:** Reads supported workbook sheets and maps input columns to canonical fields.
- **Data validator:** Checks required fields, duplicate IDs, missing values, invalid capacities, unknown subjects and timetable conflicts.
- **AI adapter:** Sends only the necessary instruction and minimal metadata to the selected provider.
- **Rule parser:** Validates the AI response against supported rule names, types and allowed values.
- **Human approval:** Shows the proposed interpretation before it can affect allocation.
- **Allocation engine:** Assigns each eligible student to a valid seat based on approved hard constraints and preferences.
- **Plan validator:** Independently checks the generated allocation.
- **Exporter:** Generates downloadable workbook, printable charts and issue reports.
- **Storage/audit:** Records plan version, input file metadata, approved rules, model/provider, timestamps and validation status. Avoid logging sensitive student data or API keys.

## 5. UI pages and expected behaviour

Use a left sidebar for navigation and a visible workflow status indicator. Pages should guide the coordinator from data import to approved seating output.

### Page 1 — Dashboard

**Purpose:** Show the current examination planning status.

Display:
- Number of imported students.
- Number of examinations/sessions.
- Number of available rooms and total usable capacity.
- Data issues requiring review.
- Current plan status: Draft, Needs Review, Ready to Allocate, Validated, or Exported.
- Recent plans.

Actions:
- `Create New Seating Plan`
- `Resume Draft`
- `View Previous Plans`

Do not show fabricated metrics when no data has been uploaded.

### Page 2 — Upload & Map Excel

**Purpose:** Import institutional data without requiring a fixed workbook layout.

Inputs:
- Student roster workbook.
- Room/capacity workbook or room sheet.
- Examination schedule and subject information.
- Optional seat layout file if the institution supplies one.

Workflow:
1. Upload `.xlsx` or `.csv`.
2. List workbook sheets and detected columns.
3. Suggest mappings to canonical fields.
4. Let the coordinator confirm mappings.
5. Preview rows before importing.
6. Run validation and show row-level errors.
7. Allow correction in the source file and re-upload, or edit supported fields in the UI.

Minimum canonical student fields:
- `student_id` — unique institutional identifier.
- `department`
- `academic_year` or `cohort`
- `exam_id` or exam-session mapping
- `subject_id` / official paper code, where applicable.

Minimum canonical room fields:
- `room_id`
- `capacity`
- `available_sessions`
- Optional accessibility or seating-layout metadata.

Minimum examination fields:
- `exam_id`
- `subject_id`
- `session_id` / date and time
- Any verified mapping for common papers across departments.

Rules:
- Do not assume similar subject names are the same paper.
- Exact official subject codes and institutional mappings take precedence.
- AI may suggest a possible match, but ambiguous matches require coordinator confirmation.
- Never silently drop invalid rows.

### Page 3 — Data Quality & Conflict Review

**Purpose:** Show issues that must be corrected before allocation.

Issue categories:
- Missing required fields.
- Duplicate student identifiers.
- Unknown or ambiguous subject mappings.
- Student assigned to multiple simultaneous exams.
- Room capacity missing or invalid.
- Exam session has more students than available seats.
- Inconsistent date/time or room availability.
- Unsupported seating rule.

Each issue should include:
- Severity: Error, Warning, or Information.
- Affected row/record reference.
- Explanation.
- Recommended correction.
- Status: Open, Resolved, or Accepted with reason.

**Blocking errors** prevent allocation. Warnings can proceed only under defined institutional policy.

### Page 4 — AI Seating Instructions

**Purpose:** Allow the coordinator to explain seating preferences in ordinary language.

Example instruction:

> “For this session, keep students writing the same paper from sitting adjacent where feasible. Prioritize accessibility requirements and use the fewest rooms possible.”

Controls:
- Free-text instruction box.
- Provider selector, for example Gemini API or Groq, based on configured credentials.
- `Interpret Instructions` button.
- Display structured rules, assumptions and unresolved questions.
- `Edit Rules`, `Ask for Clarification`, `Approve Rules` actions.

The AI should return a fixed schema, for example:

```json
{
  "rules": [
    {
      "name": "same_paper_non_adjacent",
      "type": "preference",
      "enabled": true,
      "priority": 70
    },
    {
      "name": "accessible_seating",
      "type": "hard_constraint",
      "enabled": true
    },
    {
      "name": "minimize_rooms_used",
      "type": "optimization_objective",
      "enabled": true
    }
  ],
  "clarification_required": false,
  "questions": []
}
```

This is an illustrative schema; implement the final schema as typed Pydantic models.

Supported rule examples:
- Separate students writing the same official paper when feasible.
- Avoid adjacent seats for specified groups.
- Prefer fewer rooms used.
- Use only rooms available for the session.
- Honour approved accessibility requirements.
- Prefer a specified room order.

Important:
- AI can select only from a predefined allow-list of supported rules.
- The model must not create executable Python code, SQL, arbitrary functions or unrecognized rule names.
- A rule is not active until it passes schema validation and is approved.
- A natural-language preference must not be treated as a hard constraint without confirmation.
- Never send the entire student roster or direct identifiers to the model when aggregate metadata is sufficient.
- If the API is unavailable, the coordinator must still be able to choose rules through a manual form.

### Page 5 — Room & Seat Configuration

**Purpose:** Configure the physical resources available for an examination session.

Features:
- Select rooms available for the session.
- Review usable capacity per room.
- Optionally define rows, columns, blocked seats, accessible seats and seat adjacency.
- Set room priority/order.
- Show total demand versus total usable capacity.

If only room capacities are available, the initial version may use a simple seat-index representation. Do not claim that physical adjacency is guaranteed unless the seat layout and adjacency relationships are defined.

### Page 6 — Generate Seating Plan

**Purpose:** Execute the allocation engine with approved data and rules.

Before running:
- Confirm imported data passed required checks.
- Confirm rules have been approved.
- Confirm selected rooms and sessions.
- Show demand, capacity and key constraints.

Actions:
- `Generate Plan`
- `Try Another Valid Arrangement`
- `Return to Rules`

The allocation engine should:
1. Group students by examination session and verified paper identity.
2. Build eligible room/seat candidates.
3. Enforce hard constraints.
4. Optimize preferences where possible.
5. Return a reproducible plan and a clear status.

Use a simple deterministic algorithm for the first prototype if the constraints are limited. Consider Google OR-Tools CP-SAT when there are many interacting constraints or optimization objectives.

Do not use an LLM to choose individual seats.

### Page 7 — Results & Validation

**Purpose:** Review the generated plan before publishing it.

Display:
- Overall validation status.
- Assigned student count and unassigned count.
- Duplicate assignment count.
- Capacity violations.
- Hard-rule violations.
- Preferences satisfied and not satisfied.
- Rooms used and unused.
- Any unresolved infeasibility.

Views:
- Room-wise seating chart.
- Search by student ID/register number.
- Filter by exam, session, department and room.
- Printable room view.

The independent validator must verify at minimum:
- Every eligible student is assigned exactly once.
- No seat is assigned to more than one student.
- No room exceeds usable capacity.
- Every assignment belongs to the correct exam/session.
- All hard constraints are satisfied.
- Any unmet preference is reported.

A plan with a hard-rule violation must not be marked `Validated`.

### Page 8 — Export & Publish

**Purpose:** Produce institution-ready outputs.

Excel workbook should include:
- `Room Summary`
- `Seating Plan`
- `Student Lookup`
- `Unassigned Students`
- `Validation Report`
- `Approved Rules` (optional)

Recommended seating columns:
- `exam_id`
- `session`
- `room_id`
- `seat_label`
- `student_id` or institutional register number
- `department`
- `subject_id`

Export options:
- `Download Excel`
- `Download PDF`
- `Print Room Chart`

Use a configurable privacy policy for exports. Do not include unnecessary personal fields.

Only validated plans should be publishable by default. If an institution allows a provisional export, mark it prominently as **UNVALIDATED — NOT FOR FINAL USE**.

### Page 9 — Plan History & Settings

**Purpose:** Reopen prior plans and manage configuration.

History:
- Plan name and session.
- Creation timestamp.
- Status.
- Approved rule-set version.
- Allocation algorithm version.
- AI provider/model identifier (when used).
- Validation summary.
- Export timestamps.

Settings:
- API provider and model selection.
- API key configured status (never display the key).
- Default seating policies.
- Room/seat conventions.
- Export preferences.
- Retention and access settings.

Store API keys in environment variables or a secrets manager, never in source control or the database as plain text.

## 6. Excel workbook support

The application should accept varied column names through explicit mapping rather than assuming one fixed workbook layout.

Examples of possible mappings:
- `Register No`, `Register Number`, `Roll No` → `student_id`
- `Dept`, `Department Name` → `department`
- `Subject Code`, `Paper Code` → `subject_id`
- `Hall`, `Room Number` → `room_id`
- `Strength`, `Capacity` → `capacity`

Column matching may use normalized exact matches and aliases first. AI may suggest a mapping only when deterministic mapping is uncertain; the coordinator confirms it.

Import requirements:
- Accept `.xlsx` and `.csv`.
- Enforce file-size and row-count limits.
- Validate extension and file contents.
- Treat spreadsheet values as untrusted input.
- Reject formulas or external links where not needed.
- Preserve source filename, import time and sheet name as metadata.
- Provide a downloadable error report with row numbers.
- Never silently overwrite an existing approved plan.
- Support re-importing a corrected workbook and show what changed.

Suggested sample workbook:
- `Students` sheet: student ID, department, year/cohort, subject/paper code, exam/session ID.
- `Rooms` sheet: room ID, capacity, available session(s).
- `Exams` sheet: exam ID, subject ID, session/date/time.
- Optional `SeatLayout` sheet: room ID, seat label, row, column, usable/blocked status, accessibility flag.

Official codes and verified institutional mappings must be treated as the source of truth.

## 7. AI API integration

### Provider design

Create one interface so providers can be switched without changing UI or allocation logic.

```python
class RuleInterpreter:
    def interpret(self, instruction: str, context: dict) -> dict:
        """Return a structured, schema-validated rule proposal."""
        raise NotImplementedError
```

Suggested modules:
- `ai/base.py` — provider interface.
- `ai/gemini_provider.py` — Gemini implementation.
- `ai/groq_provider.py` — optional alternative provider.
- `ai/prompts.py` — versioned system instructions.
- `schemas/rules.py` — Pydantic output models.
- `services/rule_service.py` — validation and approval workflow.

Development choice:
- Start with a free-tier API if available for the chosen model/account.
- Keep provider, model name and limits configurable.
- Treat free tiers as best-effort, not guaranteed production capacity.
- Include manual rule configuration if the API fails or quota is exhausted.
- Do not assume a provider’s model supports every structured-output feature; verify against current API documentation.

### Request flow

1. Coordinator enters an instruction.
2. Application sends the instruction and minimal de-identified context.
3. API returns proposed structured rules.
4. Pydantic validates types and schema.
5. Application checks rule names and values against the allow-list.
6. UI shows the proposal and asks for coordinator approval.
7. Approved rules are stored with a version.
8. Allocation engine consumes only approved rules.

### Security and privacy

- Use synthetic data during development.
- Do not send names, register numbers, contact details or complete rosters unless strictly necessary and institutionally approved.
- Review provider data-use, retention, region and education-data requirements before production.
- Use environment variables, such as `GEMINI_API_KEY`, for credentials.
- Add timeouts, retry with backoff, rate-limit handling and safe error messages.
- Redact secrets from logs.
- Never put API keys in browser-side code, notebooks committed to the repository, or the README.

## 8. Seating rules: hard constraints vs preferences

Separate these categories explicitly.

### Hard constraints

Must never be violated in a validated plan:
- Every eligible student is assigned exactly once.
- No duplicate seat occupancy.
- Room capacity is not exceeded.
- Assignment matches the correct exam/session.
- Approved accessibility constraints are honoured.
- Institutional mandatory rules are satisfied.

### Preferences / optimization objectives

Optimize when feasible:
- Students writing the same paper are not adjacent.
- Specified cohorts are separated according to policy.
- Use fewer rooms.
- Respect room priority.
- Balance room occupancy.

The institution decides which policies are hard constraints and which are preferences. The AI can interpret a request but cannot unilaterally change its severity. If hard constraints conflict, report infeasibility and request administrator action.

## 9. Suggested repository structure

```text
exam-seat-generator/
├── app.py
├── pages/
│   ├── 1_Dashboard.py
│   ├── 2_Upload_Excel.py
│   ├── 3_Data_Quality.py
│   ├── 4_AI_Instructions.py
│   ├── 5_Room_Configuration.py
│   ├── 6_Generate_Plan.py
│   ├── 7_Results_Validation.py
│   ├── 8_Export_Publish.py
│   └── 9_History_Settings.py
├── src/
│   ├── ai/
│   │   ├── base.py
│   │   ├── gemini_provider.py
│   │   ├── groq_provider.py
│   │   └── prompts.py
│   ├── schemas/
│   │   ├── input_data.py
│   │   └── rules.py
│   ├── importers/
│   │   └── excel_importer.py
│   ├── validators/
│   │   ├── input_validator.py
│   │   └── plan_validator.py
│   ├── allocation/
│   │   ├── engine.py
│   │   └── constraints.py
│   ├── exporters/
│   │   ├── excel_exporter.py
│   │   └── pdf_exporter.py
│   ├── services/
│   │   ├── plan_service.py
│   │   └── rule_service.py
│   └── storage/
│       └── repository.py
├── templates/
│   └── sample_exam_workbook.xlsx
├── tests/
│   ├── test_excel_import.py
│   ├── test_rule_schema.py
│   ├── test_allocation.py
│   └── test_plan_validation.py
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

This is a target structure; create files incrementally as features are implemented.

## 10. Environment configuration

Example `.env.example`:

```dotenv
AI_PROVIDER=gemini
AI_MODEL=
GEMINI_API_KEY=
GROQ_API_KEY=

DATABASE_URL=sqlite:///./exam_seat_generator.db
MAX_UPLOAD_MB=20
```

- Never commit the real `.env`.
- Keep `.env.example` free of real credentials.
- If no AI key is configured, manual rule selection must remain usable.

## 11. Development setup

Example setup for Windows PowerShell:

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
streamlit run app.py
```

Example minimal dependencies (pin compatible versions after testing):

```text
streamlit
pandas
openpyxl
pydantic
python-dotenv
google-genai
groq
ortools
reportlab
pytest
```

Only install provider SDKs that are actually used. OR-Tools and PDF generation can be added when those features are implemented.

## 12. Testing and acceptance criteria

Before a pilot, test at least the following cases with synthetic data:

1. A valid student roster imports with correct column mappings.
2. Missing required fields block allocation and identify the row.
3. Duplicate student IDs are detected.
4. Two departments sharing a paper are matched only by verified paper ID or approved mapping.
5. Similar subject names with different official codes are not automatically merged.
6. AI output with invalid JSON, unknown rules or wrong types is rejected.
7. AI quota exhaustion falls back to manual rule selection.
8. Hard constraints are enforced by the allocator and independently verified.
9. Infeasible capacity is reported clearly.
10. Every student is assigned exactly once in a valid plan.
11. Exported Excel workbook contains expected sheets and row counts.
12. Re-running the same inputs and configuration produces a reproducible result where the selected algorithm supports deterministic output.
13. Secrets and direct identifiers do not appear in application logs.
14. A plan cannot be marked validated if hard-rule checks fail.

## 13. Implementation order

1. Build Excel import, canonical data mapping and validation.
2. Implement the simplest deterministic allocation and independent plan validator.
3. Build room-wise result views and Excel export.
4. Add manual rule controls.
5. Add one AI provider to interpret natural-language instructions into the fixed schema.
6. Add human review, approval and audit history.
7. Add the optional second provider and local model only after the first workflow is stable.
8. Pilot with one department and synthetic or approved test data; compare coordinator preparation time and correction rates with the manual process.

## 14. Success measures

Measure:
- Time from workbook upload to validated plan.
- Number of manual corrections required.
- Percentage of valid plans generated on the first attempt.
- Number of hard-rule violations (target: zero in validated plans).
- AI schema-valid response rate.
- Correct interpretation rate on a fixed test set.
- API cost per planning request.
- Successful completion rate when AI is unavailable.

## 15. Final design decision

For the initial release:

- Use Streamlit for a simple multi-page UI.
- Support Excel/CSV import and downloadable Excel output.
- Start with one cloud AI provider using a free development tier if available.
- Keep a provider adapter so another API can be added later.
- Use AI only for natural-language interpretation and uncertain mapping suggestions.
- Keep seating allocation and all hard validation deterministic.
- Require coordinator approval of AI-proposed rules.
- Keep manual rule selection as a complete fallback.

This provides a small, maintainable institutional pilot without unnecessary infrastructure or pretending that AI replaces a reliable allocation algorithm.
