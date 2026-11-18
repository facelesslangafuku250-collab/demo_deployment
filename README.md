# SERC: Security Event Response Copilot

A web app that helps a security analyst understand a Wazuh alert. It finds relevant passages in official
sources (MITRE ATT&CK, NIST CSF 2.0, CERT-SG IRM playbooks, NIST SP 800-61), builds suggested next steps from them,
cites the source of every step, and checks every statement before showing how far it can be trusted.

It is based on the paper *Enhancing Security Operations Center: Wazuh Security Event Response with
Retrieval-Augmented-Generation-Driven Copilot* (Ismail et al., Sensors 2025, 25, 870, doi:10.3390/s25030870).
SERC re-creates its retrieval pipeline and adds an evidence validator, a human-approval step and flexible input.

## What is new compared with the paper

| Paper | This project |
|---|---|
| One alert shown with generated guidance | Every step carries a citation (E1, E2 ...) and links to the source passage |
| Quality measured offline against reference playbooks | Each statement is validated at runtime and labelled Supported, Partially supported, Conflicting, Not enough evidence or Needs review |
| No analyst control | Disruptive steps are flagged "needs analyst approval"; approvals and decisions are saved and exported |
| Wazuh JSON only | Wazuh JSON, pasted log lines, `.log` / `.txt` / `.csv` files, and screenshots (live mode) |
| Three collections | Three knowledge sources (MITRE, NIST, incident response) searched separately, de-duplicated and balanced |
| Needs LLM access | Works fully offline (every step is a sentence taken from a cited source); a live model is optional |

## How it works

1. **Input:** the alert is normalised into one Wazuh-style structure. Details guessed from plain text are shown for the analyst to confirm.
2. **Retrieval:** if the alert names a MITRE technique, its records are fetched directly; then each knowledge source is searched by meaning (BAAI/bge-large-en-v1.5, cosine similarity, Qdrant).
3. **Guidance:** offline mode picks action sentences from the retrieved passages. Live mode asks an LLM to write steps from the same passages and cite them.
4. **Validation:** facts must appear in the alert; MITRE mappings must come from the Wazuh rule or a retrieved record; steps must be grounded in the passage they cite; invented IP addresses, "is compromised" claims and citations of passages that were never retrieved are flagged.
5. **Analyst:** approves steps, accepts or rejects statements, asks follow-up questions, downloads a report.

## Setup (Windows, Python 3.10+)

1. Put the downloaded sources in `data_sources/`: `enterprise-attack.json`, `csf2.xlsx`, `NIST.SP.800-61r2.pdf` / `nist.sp.800-61r3.pdf`, and the CERT-SG `IRM` folder (PDFs).
2. `pip install -r backend/requirements.txt`
3. Copy `backend/.env.example` to `backend/.env`. Keep `QDRANT_PATH` **outside OneDrive**.
4. Check parsing without embedding anything: `python backend/scripts/ingest_sources.py --dry-run`
5. Build the knowledge base (server stopped): `python backend/scripts/ingest_sources.py --quick --recreate`
   `--quick` covers the 29 ATT&CK techniques used by the sample alerts. Drop `--quick` for all of ATT&CK (slower). Re-running without `--recreate` only adds what is missing.
6. `python backend/main.py`, then open http://127.0.0.1:8000

Only one program can use the embedded Qdrant folder at a time: stop the server before ingesting.

## Using it

Paste an alert, upload a file or screenshot, or choose a sample. Check the details, press **Analyze**, then use the tabs:
Report, MITRE ATT&CK, Evidence, Verification and Ask a question. The "Technical details" switch shows scores and IDs.

## Knowledge sources (cite these)

| Source | Used for |
|---|---|
| MITRE ATT&CK Enterprise, STIX data (github.com/mitre-attack/attack-stix-data); version: _fill in from the ingestion log line "MITRE collection"_ | Technique descriptions and mitigations |
| NIST Cybersecurity Framework 2.0, reference-tool export (csrc.nist.gov) | Functions, categories, subcategories, implementation examples |
| CERT Societe Generale, Incident Response Methodologies (github.com/certsocietegenerale/IRM); see its LICENSE | Incident-handling playbooks |
| NIST SP 800-61 Rev. 3 (Rev. 2 optional) | General incident-response guidance |

Downloaded 14 November 2026. Sample alerts in `backend/data/sample_alerts.json` are **synthetic** test input, not knowledge.

## Limitations (please read)

- Validation shows whether a statement is **traceable** to the alert or a cited source. It does not prove the advice is correct or safe for a given environment. Grounding is a word-overlap check, not full semantic verification.
- Offline guidance is only as good as the retrieved passages. Weak matches are labelled, and the answer can be incomplete.
- The ingested ATT&CK file has no detection text for techniques, so detection passages are absent. Its tactic names may differ from older Wazuh rule names.
- Live-model and screenshot reading are implemented but were not tested against the real services. Offline mode is the tested path.
- The paper's evaluation (cosine similarity and BERTScore against SOCFortress playbooks, 12 attack scenarios) was **not** reproduced here.
- Severity bands (Low, Medium, High, Critical) are SERC's own display labels, not an official Wazuh scale.
- Alerts are not sent anywhere in offline mode. In live mode, alert text and retrieved passages are sent to the configured provider.

## API

`GET /api/health` · `GET /api/alerts/samples` · `GET /api/alerts/samples/{id}` · `POST /api/alerts/ingest` · `POST /api/alerts/analyze` ·
`POST /api/alerts/chat` · `POST /api/feedback` · `GET /api/feedback/{alert_id}`. Interactive docs: http://127.0.0.1:8000/docs

## Tests

`python -m pytest tests -v` (or `python tests/test_serc.py`). They use an in-memory knowledge base built from your `csf2.xlsx`.
