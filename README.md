# Authorized penetration-test agent

LangGraph control-plane agent for one scoped engagement. It checks authorization, drafts a plan, pauses for a named approver, drops out-of-scope notes, and writes a Markdown report.

It does not run exploits, password attacks, malware, or denial-of-service actions.

The evaluation brief is [docs/EVALUATION.md](docs/EVALUATION.md). Send that file.

## Run

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m pentest_agent.demo
```

The demo shows three cases: missing authorization, a forbidden action, and a paused plan that resumes after approval.
# pen-test-basic
