# CLAUDE.md

This file is the working constitution for Claude Code in this repository. Keep it short, practical, and focused on what the agent must do correctly for PharmaFlow AI.

PharmaFlow AI is a capstone/demo project for payer-side pharmacy benefit analysis. The system recommends clinically reasonable, risk-adjusted drug-switch opportunities using synthetic claims data, RxNorm/FDA drug mapping, NADAC-style pricing, and agentic analysis. It is not a medical device, not clinical decision support for live patient care, and must never process real PHI unless the repo is explicitly upgraded for compliance.

---

## Project Goal

Build a deployed multi-agent payer dashboard that helps an insurance company answer:

> “Where can we reduce pharmacy spend without increasing total cost of care or harming adherence?”

The demo should prioritize:
- Clear ROI story for insurers.
- Safe drug-alternative reasoning.
- Transparent assumptions.
- Synthetic but realistic payer-style data.
- Reproducible local and deployed runs.
- A simple, convincing UI over unnecessary backend complexity.

---

## Non-Negotiable Guardrails

### Healthcare Safety

- Treat all patient-level data in this repo as synthetic unless explicitly documented otherwise.
- Do not introduce, request, or store real patient identifiers, PHI, insurance member IDs, addresses, phone numbers, or clinical notes.
- Do not present recommendations as final medical advice.
- Phrase outputs as payer-facing review suggestions, for example:
  - “Potential switch candidate for pharmacist/clinician review.”
  - “Estimated savings opportunity with clinical risk flags.”
- If clinical equivalence is uncertain, surface uncertainty instead of forcing a recommendation.
- Always preserve the distinction between:
  - generic equivalent,
  - therapeutic alternative,
  - formulary alternative,
  - lower-cost NDC/package,
  - clinically risky switch.

### Data Integrity

- Do not fabricate source-backed claims inside the code or UI.
- Synthetic data is allowed, but label it clearly as synthetic.
- Keep generated data realistic enough for a payer workflow:
  - members,
  - claims,
  - prescriptions,
  - NDC/RxCUI mappings,
  - drug costs,
  - formularies,
  - adherence signals,
  - medical event risk,
  - pharmacy access / SDoH fields.
- Every savings number shown in the UI should be traceable to a formula or transformation in code.
- If a data source is unavailable, use deterministic fallback data and clearly mark the fallback.

### Agent Behavior

- Agents should be modular and narrow.
- Prefer simple tool calls and typed functions over vague LLM reasoning.
- LLMs may summarize, explain, and rank opportunities, but core savings/risk calculations must be deterministic Python.
- Any agent output used in the UI must include:
  - recommendation,
  - estimated gross savings,
  - estimated risk-adjusted savings,
  - uncertainty or confidence,
  - reason codes,
  - safety/access flags.

---

## System Architecture

The intended architecture is:

```text
Frontend Dashboard
        |
FastAPI Backend
        |
Agent Orchestrator
        |
+----------------------+----------------------+----------------------+----------------------+
| Librarian Agent      | Auditor Agent        | Clinician Agent      | Social Navigator     |
| Drug mapping         | Cost/spread analysis | Risk-adjusted TCOC   | Access/adherence     |
+----------------------+----------------------+----------------------+----------------------+
        |
Synthetic + Public Data Layer
```

### Agent Responsibilities

#### Librarian Agent

Responsible for drug identity and mapping.

Use for:
- RxNorm / RxCUI mapping.
- Brand to generic mapping.
- Ingredient and strength matching.
- FDA Orange Book / DailyMed style metadata when available.
- Distinguishing true generic equivalents from broader therapeutic alternatives.

Avoid:
- Recommending a switch only because names look similar.
- Treating different dosage forms or strengths as interchangeable without flags.

Expected output fields:
- `source_drug`
- `candidate_alternative`
- `equivalence_type`
- `rxcui`
- `ndc`
- `ingredient`
- `strength`
- `dose_form`
- `mapping_confidence`
- `mapping_reason`

#### Auditor Agent

Responsible for price and payer economics.

Use for:
- NADAC-style unit cost comparison.
- Synthetic PBM spread estimation.
- Formulary tier analysis.
- Gross savings calculation.
- Audit-readiness explanations.

Avoid:
- Assuming rebate amounts are known unless present in synthetic data.
- Hiding whether savings are gross, net, or risk-adjusted.

Expected output fields:
- `current_unit_cost`
- `alternative_unit_cost`
- `quantity`
- `days_supply`
- `gross_savings`
- `spread_estimate`
- `net_savings_assumption`
- `audit_reason`

#### Clinician Agent

Responsible for risk-adjusted total cost of care.

Use for:
- Switch failure probability.
- Medical event risk adjustment.
- Prior authorization / contraindication style flags in synthetic data.
- Bayesian or probabilistic uncertainty ranges.

Avoid:
- Making diagnosis or treatment claims beyond the synthetic features.
- Removing clinically important risk flags to improve ROI.

Expected output fields:
- `clinical_risk_score`
- `switch_failure_probability`
- `expected_medical_cost_delta`
- `risk_adjusted_savings`
- `credible_interval_low`
- `credible_interval_high`
- `clinical_reason_codes`

#### Social Navigator Agent

Responsible for access and adherence feasibility.

Use for:
- Pharmacy access score.
- Distance or pharmacy desert proxy.
- Refill friction.
- Preferred chain availability.
- Adherence risk flags.

Avoid:
- Recommending a lower-cost drug that is inaccessible to the member.
- Treating theoretical savings as realized savings when access risk is high.

Expected output fields:
- `pharmacy_access_score`
- `adherence_risk_score`
- `preferred_pharmacy_available`
- `access_override`
- `access_reason`

---

## Core Calculation Rules

The project should separate deterministic calculations from LLM narration.

### Gross Pharmacy Savings

Use this structure unless a better documented formula exists:

```text
gross_savings =
    (current_unit_cost - alternative_unit_cost) * normalized_quantity
```

Where:
- unit cost should be normalized to comparable units,
- package size differences must be handled explicitly,
- quantity and days supply must be validated.

### Risk-Adjusted Savings

Use this structure unless a better documented formula exists:

```text
risk_adjusted_savings =
    gross_savings
    - expected_medical_cost_delta
    - expected_adherence_penalty
```

Where:

```text
expected_medical_cost_delta =
    switch_failure_probability * estimated_downstream_event_cost
```

### Recommendation Bands

Default bands:

```text
Recommend       risk_adjusted_savings > 0 and clinical/access risk is low
Review          savings positive but clinical/access uncertainty is meaningful
Do Not Switch   risk-adjusted savings <= 0 or safety/access flags are high
```

Do not hard-code these thresholds in multiple places. Put them in one config or scoring module.

---

## Repository Conventions

Expected structure:

```text
.
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── agents/
│   │   ├── api/
│   │   ├── core/
│   │   ├── data/
│   │   ├── models/
│   │   └── services/
│   ├── tests/
│   └── requirements.txt or pyproject.toml
├── frontend/
│   ├── src/
│   └── package.json
├── data/
│   ├── synthetic/
│   ├── raw/
│   └── processed/
├── scripts/
├── docs/
├── AGENTS.md
└── CLAUDE.md
```

If the actual repo differs, inspect the tree first and preserve the existing structure.

---

## Python / Backend Rules

- Prefer Python 3.11+ unless the repo specifies otherwise.
- Use FastAPI for backend routes.
- Use Pydantic models for request/response schemas.
- Keep business logic out of route handlers.
- Put scoring formulas in reusable service modules.
- Put agent prompts/config in dedicated agent files.
- Use explicit types for all public functions.
- Prefer pure deterministic functions for:
  - cost calculations,
  - risk scoring,
  - data validation,
  - recommendation classification.

### Backend Commands

Before changing commands, inspect `pyproject.toml`, `requirements.txt`, `Makefile`, and README.

Common commands:

```bash
# Install backend dependencies
cd backend && pip install -r requirements.txt

# Run API locally
cd backend && uvicorn app.main:app --reload

# Run tests
cd backend && pytest -q
```

If a command fails because the project uses a different package manager, adapt to the repo’s existing config and update this file only if the change is broadly useful.

---

## Frontend Rules

- Keep the dashboard simple and demo-ready.
- Prioritize payer-facing clarity over visual complexity.
- Every recommendation card/table should show:
  - current drug,
  - alternative drug,
  - gross savings,
  - risk-adjusted savings,
  - clinical risk,
  - access risk,
  - recommendation band,
  - reason codes.
- Avoid showing unexplained “AI score” values.
- Prefer readable tables, filters, and drilldowns.

Common commands:

```bash
# Install frontend dependencies
cd frontend && npm install

# Run frontend locally
cd frontend && npm run dev

# Build frontend
cd frontend && npm run build
```

---

## Data Rules

### Synthetic Claims Data

Synthetic payer data should include enough fields to support realistic analysis:

```text
member_id
age_band
sex
zip3
plan_id
claim_id
drug_name
brand_generic_flag
ndc
rxcui
quantity
days_supply
fill_date
paid_amount
member_cost_share
pharmacy_id
prescriber_id
diagnosis_group
adherence_score
prior_switch_failure_flag
estimated_event_cost
```

Use stable random seeds for generated data.

### Public Drug / Price Data

Use public data when feasible, such as:
- RxNav/RxNorm-style mappings.
- FDA Orange Book style equivalence concepts.
- DailyMed-style drug metadata.
- NADAC-style unit cost data.
- CMS-style utilization data.

If public APIs or downloads are unavailable during development, create a small deterministic fixture that mirrors the expected schema.

### Data Documentation

For every dataset, document:
- source or synthetic generator,
- schema,
- refresh process,
- known limitations,
- whether it is safe for demo use.

Put detailed dataset notes in `docs/data.md`, not in this file.

---

## LLM and Agent Rules

- Do not let LLM text override deterministic calculations.
- LLM summaries must cite structured fields from the agent outputs.
- Keep prompts short and task-specific.
- Prefer structured JSON outputs with Pydantic validation.
- If an LLM response fails validation, retry once with the validation error.
- If it fails again, return a deterministic fallback explanation.

### Good Agent Output

```json
{
  "recommendation": "Review",
  "reason": "Alternative has lower unit cost, but member has moderate adherence risk and limited pharmacy access.",
  "gross_savings": 420.50,
  "risk_adjusted_savings": 175.25,
  "confidence": "medium",
  "reason_codes": ["LOWER_NADAC_COST", "MODERATE_ACCESS_RISK", "CLINICAL_REVIEW_REQUIRED"]
}
```

### Bad Agent Output

```json
{
  "recommendation": "Switch immediately",
  "reason": "The generic is cheaper and should work."
}
```

---

## Testing Expectations

Add or update tests when changing:
- savings formulas,
- risk scoring,
- drug mapping,
- recommendation thresholds,
- API response schemas,
- synthetic data generation.

Minimum useful tests:
- brand-to-generic mapping works for known fixtures,
- unit cost normalization is correct,
- risk-adjusted savings subtracts expected downstream cost,
- high clinical risk blocks automatic recommend,
- low pharmacy access creates review/override behavior,
- API returns valid response schema.

Run tests before finalizing backend changes:

```bash
cd backend && pytest -q
```

For frontend changes, run:

```bash
cd frontend && npm run build
```

---

## Demo / Capstone Priorities

For the May 4 capstone deadline, prioritize in this order:

1. End-to-end demo flow works.
2. Synthetic data is realistic and clearly documented.
3. Dashboard clearly explains payer savings and risk.
4. Agents produce structured, inspectable outputs.
5. Scoring formulas are deterministic and testable.
6. Deployment works from a clean checkout.
7. README explains setup, live URL, architecture, class concepts, and limitations.

Avoid spending time on:
- perfect clinical modeling,
- large-scale infrastructure,
- complex PDF parsing unless needed for the demo,
- excessive agent autonomy,
- unsupported real-world compliance claims.

---

## Documentation Rules

Keep this file high-level. Do not turn it into a full manual.

Use pointers like this:

```text
For complex NADAC normalization issues, see docs/data.md.
For deployment troubleshooting, see docs/deployment.md.
For scoring formula details, see docs/scoring.md.
For agent prompt design and validation, see docs/agents.md.
```

Do not embed long external docs here. Explain when Claude should read them.

---

## README Requirements

The README should include:
- project overview,
- target user,
- problem statement,
- architecture diagram or text architecture,
- setup instructions,
- local run commands,
- deployment URL,
- data sources,
- synthetic data disclaimer,
- agent descriptions,
- scoring methodology,
- class concepts used with file references,
- known limitations,
- demo script.

---

## Deployment Rules

- Prefer simple deployment over complex infrastructure.
- Keep environment variables documented in `.env.example`.
- Never commit real secrets.
- If using Cloud Run, keep build and deploy commands in README or `docs/deployment.md`.
- Health check endpoint should exist, for example:

```text
GET /health
```

Expected response:

```json
{
  "status": "ok"
}
```

---

---

## GCP / Cloud Run Deployment Rules

Use Google Cloud Platform for the capstone deployment. Prefer the simplest reproducible path that works from a clean checkout.

### Default GCP Target

Use these defaults unless the user or repo specifies otherwise:

```bash
GCP_PROJECT_ID=<your-project-id>
GCP_REGION=us-central1
CLOUD_RUN_SERVICE=pharmaflow-ai
```

The app should expose:

```text
GET /health
```

and return:

```json
{
  "status": "ok"
}
```

Cloud Run services must listen on the `PORT` environment variable injected by Cloud Run. Do not hard-code port `8000` for production startup.

### One-Command Source Deploy

For a fast capstone demo, prefer Cloud Run source deployment from the backend service directory if the repo is structured that way:

```bash
gcloud config set project $GCP_PROJECT_ID

gcloud services enable run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  logging.googleapis.com

cd backend

gcloud run deploy $CLOUD_RUN_SERVICE \
  --source . \
  --region $GCP_REGION \
  --allow-unauthenticated \
  --set-env-vars DATA_MODE=synthetic,USE_LLM=false
```

Use this path when:
- the backend can run as a standalone FastAPI service,
- the frontend is separate or already hosted elsewhere,
- speed and reliability matter more than custom infrastructure.

### Docker + Artifact Registry Deploy

Use this path when the repo has a Dockerfile or when source deploy is not reproducible.

```bash
gcloud config set project $GCP_PROJECT_ID

gcloud services enable run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  logging.googleapis.com

gcloud artifacts repositories create pharmaflow-repo \
  --repository-format=docker \
  --location=$GCP_REGION \
  --description="PharmaFlow AI container images"
```

Build and push with Cloud Build:

```bash
IMAGE_URI=$GCP_REGION-docker.pkg.dev/$GCP_PROJECT_ID/pharmaflow-repo/pharmaflow-ai:latest

gcloud builds submit . --tag $IMAGE_URI

gcloud run deploy $CLOUD_RUN_SERVICE \
  --image $IMAGE_URI \
  --region $GCP_REGION \
  --allow-unauthenticated \
  --set-env-vars DATA_MODE=synthetic,USE_LLM=false
```

Use this path when:
- there is a production Dockerfile,
- frontend and backend are packaged together,
- dependencies require system packages,
- the deployment must match local Docker behavior.

### Frontend Deployment Options

If the frontend is separate, use one of these simple options:

1. Build frontend static assets and serve them from the backend if the repo already supports it.
2. Deploy frontend separately using Vercel, Netlify, Firebase Hosting, or Cloud Run.
3. For the capstone, a backend-hosted simple dashboard is preferred over a fragile multi-service deployment.

Do not create a complex Kubernetes, Terraform, or multi-service setup unless the repo already uses it.

### GCP Environment Variables

Required variables should be documented in `.env.example`.

Recommended demo defaults:

```bash
DATA_MODE=synthetic
USE_LLM=false
MODEL_NAME=
CORS_ORIGINS=*
```

Optional LLM variables:

```bash
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GOOGLE_API_KEY=
```

Never commit real API keys. If secrets are needed in Cloud Run, use Secret Manager or set them through Cloud Run environment configuration rather than committing them to the repo.

### Deployment Validation

After every deploy, verify:

```bash
SERVICE_URL=$(gcloud run services describe $CLOUD_RUN_SERVICE \
  --region $GCP_REGION \
  --format='value(status.url)')

curl "$SERVICE_URL/health"
```

Then test at least one real demo endpoint, for example:

```bash
curl "$SERVICE_URL/api/recommendations"
```

If the endpoint path differs, inspect FastAPI routes before changing the command.

### GCP Troubleshooting

If deployment fails:
- Check whether the active project is correct with `gcloud config get-value project`.
- Check whether required APIs are enabled.
- Check Cloud Run logs before changing code.
- Confirm the app listens on `0.0.0.0:$PORT`.
- Confirm `requirements.txt`, `pyproject.toml`, or Dockerfile includes all runtime dependencies.
- Confirm synthetic data files are included in the deployed image or generated at startup.

Useful commands:

```bash
gcloud run services logs read $CLOUD_RUN_SERVICE \
  --region $GCP_REGION \
  --limit 100

gcloud run services describe $CLOUD_RUN_SERVICE \
  --region $GCP_REGION
```

For advanced deployment troubleshooting, create or read `docs/deployment.md`. Keep only the 80% path here.

## Environment Variables

Document expected variables in `.env.example`.

Likely variables:

```bash
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
MODEL_NAME=
USE_LLM=false
DATA_MODE=synthetic
DATABASE_URL=
CORS_ORIGINS=
```

Default local development should work with `USE_LLM=false` and synthetic fixtures.

---

## Code Style

- Make code boring, readable, and testable.
- Prefer small functions over large agent scripts.
- Avoid hidden global state.
- Avoid notebook-only logic for production paths.
- Use meaningful names:
  - `calculate_risk_adjusted_savings`
  - `normalize_unit_cost`
  - `classify_recommendation`
- Avoid vague names:
  - `process`
  - `magic_score`
  - `run_ai`
  - `final_output`

---

## When Unsure

Follow this order:

1. Preserve healthcare safety.
2. Preserve deterministic calculations.
3. Preserve synthetic-data clarity.
4. Preserve demo reliability.
5. Preserve concise documentation.

If a requested change makes the demo less safe, less traceable, or less reproducible, implement a safer alternative and explain the tradeoff.

---

## AGENTS.md Sync

Keep `AGENTS.md` synchronized with this file for compatibility with other AI coding tools.

When updating this file:
- update `AGENTS.md` with the same agent-facing rules,
- avoid tool-specific instructions unless necessary,
- keep both files concise.
