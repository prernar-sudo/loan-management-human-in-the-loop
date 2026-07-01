# Loan Management System with Human-in-the-Loop (LangGraph + Ollama)

An agentic loan approval workflow that combines AI-driven analysis with human oversight at critical decision points — built using **LangGraph's `interrupt()`** mechanism and a **local LLM (Ollama / Llama 3.2)**.

---

## The Problem

The Reserve Bank of India's loan department processes hundreds of applications daily, each requiring KYC verification, credit checks, fraud screening, risk assessment, and compliance checks. The manual process is slow (5–7 days), inconsistent across officers, and offers no clean audit trail.

**The solution:** let AI agents handle routine screening and risk tiering automatically, while routing only high-risk or ambiguous cases to a human loan officer — exactly where judgment matters most.

---

## How It Works

This project uses LangGraph's `interrupt()` function to **pause** the workflow mid-execution and wait for human input, then resume exactly where it left off using `Command(resume=...)`.

```
Application Submitted
        │
        ▼
  Loan Analyser (Ollama LLM)
  checks credit, fraud, risk, compliance
        │
   ┌────┴────┐
   ▼         ▼
LOW RISK   HIGH RISK
(auto)     (needs human)
   │           │
   │      ┌────┴─────┐
   │      │ interrupt │ ◄── Loan Officer Reviews
   │      └────┬─────┘
   ▼           ▼
   Process Decision
        │
        ▼
   Audit Logger (RBI compliance trail)
        │
        ▼
       END
```

The paused workflow consumes **zero compute** while waiting — LangGraph's checkpointing system persists the full state so the graph can resume seconds, hours, or days later.

---

## Architecture

| Node | Responsibility | Key Feature |
|---|---|---|
| `loan_analyser` | Uses Ollama LLM to assess credit, fraud, risk, and RBI policy compliance | Rule-based pre-checks + LLM analysis, robust JSON parsing |
| `human_review` | Pauses the workflow and presents a review packet to a loan officer | Uses `interrupt()` |
| `auto_approve` | Automatically approves low-risk, policy-compliant loans | No human needed |
| `process_decision` | Finalizes the decision and triggers downstream actions | Converges both paths |
| `audit_logger` | Persists the full decision trace | Mandatory for compliance |

**Checkpointing:** `InMemorySaver` is required to make `interrupt()` / `Command(resume=...)` work — without it, paused state can't be saved or resumed.

---

## Tech Stack

- **[LangGraph](https://langchain-ai.github.io/langgraph/)** — stateful multi-step agent orchestration, `interrupt()` / `Command(resume=...)` HITL pattern
- **[Ollama](https://ollama.com/)** — local LLM inference (`llama3.2`)
- **LangChain** (`langchain_ollama`) — LLM integration
- **Python** (`TypedDict` for structured graph state)

---

## Routing Logic

A loan is routed to **human review** if any of the following are true:
- Loan amount exceeds the auto-approve limit (₹5 lakh)
- AI-assessed risk level is `medium` or `high`
- Any fraud/policy flags are detected (low CIBIL, high DTI, insufficient employment history, vague purpose)
- The AI's own recommendation is `review`

Otherwise, the loan is **auto-approved** by the AI.

---

## RBI Policy Parameters (Demo Values)

| Parameter | Value |
|---|---|
| Auto-approve limit | ₹5,00,000 |
| Minimum CIBIL score | 650 |
| Maximum DTI ratio | 50% |
| High-risk amount threshold | ₹20,00,000 |
| Minimum employment history | 1 year |

---

## Setup

### 1. Install and run Ollama
```bash
# Install Ollama: https://ollama.com/download
ollama pull llama3.2
ollama serve
```

### 2. Install Python dependencies
```bash
pip install langgraph langchain-ollama
```

### 3. Run the notebook
Open `loan_management_system_with_human_in_loop.ipynb` in Jupyter and run all cells sequentially. Ensure Ollama is running locally on `http://localhost:11434` before executing the LLM calls.

---

## Example Usage

**Auto-approved case (low risk):**
```python
config = {"configurable": {"thread_id": "loan-thread-001"}}
result = loan_graph.invoke({"loan_id": "LOAN-001"}, config)
```

**Human-in-the-loop case (high risk — pauses for officer input):**
```python
from langgraph.types import Command

# Step 1: Start — workflow pauses at interrupt()
config = {"configurable": {"thread_id": "loan-thread-002"}}
result = loan_graph.invoke({"loan_id": "LOAN-002"}, config)

# Step 2: Officer reviews the packet, then resumes the workflow
result = loan_graph.invoke(Command(resume="approved"), config)
# or: Command(resume="rejected: <reason>")
# or: Command(resume="query: <question for applicant>")
```

---

## Test Scenarios Included

| Loan ID | Applicant | Scenario |
|---|---|---|
| LOAN-001 | Priya Sharma | Low-risk personal loan → auto-approved |
| LOAN-002 | Rajesh Verma | Large business loan → human review → approved |
| LOAN-003 | Anil Kumar | Low CIBIL, high DTI, vague purpose → human review → rejected |
| LOAN-004 | Meena Patel | Home loan → human review → query raised for more documents |

---

## Key Learning: Why `interrupt()` Matters

Traditional agent workflows either run fully autonomously (risky for high-stakes decisions) or require constant human supervision (defeats the purpose of automation). LangGraph's HITL pattern lets the AI handle everything it's confident about, and cleanly hands off only the decisions that genuinely need a human — with the full context, analysis, and reasoning already prepared for the reviewer.

---

## Disclaimer

This is a demo/educational project using synthetic loan data. It does not connect to any real banking systems, credit bureaus, or RBI infrastructure.

---

## Author

Built by [Prerna](https://github.com/prernar-sudo) as part of an agentic AI project series exploring multi-agent systems, HITL architectures, and local LLM deployment.
