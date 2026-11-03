# Legal Contract Ticket Decision Engine: Rule-Prioritized LLM Classification with Adversarial Testing

An experimental **Agentic AI workflow** that classifies a user's message
about a contract problem into **exactly one action** from a fixed set,
using a prioritized rule prompt, strict output validation, and a
dedicated test strategy built around adversarial and ambiguous cases.

The notebook is built in stages. It first teaches scope control with
three actions, then extends the same code to all five actions with an
escalation rule, and evaluates each stage with hand-designed test
cases that document the trap each case is meant to expose.

> **Project:** Legal Contract Ticket Decision Engine\
> **Notebook:** `Legal_Contract_Agent.ipynb`\
> **Domain:** Agentic AI · LLMs · Prompt Design · Output Validation ·
> Adversarial Testing · Python\
> **Environment:** Google Colab

------------------------------------------------------------------------

## 1. Project Overview

The goal is to build a **decision engine**, not a legal advisor. For
each incoming message it chooses the next action and never gives legal
advice, explains clauses, or suggests what the user should do.

The workflow keeps the model to a single classification job and places
programmatic controls around it:

1.  Send the message to the LLM with a system prompt that defines the
    allowed actions and the order in which rules apply.
2.  Require a JSON-only reply with `action`, `confidence`, and
    `reasoning`.
3.  Extract the JSON object from the raw text.
4.  Validate that all keys exist, that the action is a real action, and
    that it is **allowed at the current stage**.
5.  Evaluate the engine with basic, adversarial, and ambiguous test
    suites that state the expected action and why.

The five actions are:

-   `CREATE_TICKET`
-   `ASK_FOR_MORE_INFO`
-   `REJECT_REQUEST`
-   `FLAG_OUT_OF_SCOPE`
-   `ESCALATE_TO_HUMAN`

The central engineering principle is:

> **A classification prompt is only as trustworthy as the tests that
> try to break it: define the rules in priority order, validate every
> output, and test the cases designed to fool the system.**

------------------------------------------------------------------------

## 2. Key Features

### Staged action sets

-   **Part 1** exposes only three actions: `REJECT_REQUEST`,
    `FLAG_OUT_OF_SCOPE`, and `ASK_FOR_MORE_INFO`, to focus on scope
    control.
-   **Part 3** unlocks all five actions, adding `CREATE_TICKET` and
    `ESCALATE_TO_HUMAN`.
-   The allowed set is passed into validation, so an action that is
    valid in the enum but not allowed at the current stage is rejected.

### Priority-ordered rule prompts

-   Rules apply in a fixed order, and the model must stop at the first
    match.
-   Part 1 order: advice request, then no contract signal, then ask
    for more information.
-   Part 3 order: advice request, then no agreement signal, then
    escalation on risk signals, then ticket creation only if all
    completeness conditions hold, otherwise ask for more information.
-   Contractual relationships are detected semantically (agreement,
    terms, partnership, MOU), not by the word "contract".

### Escalation on risk signals

-   Hidden or exploited terms, deliberate concealment, major one-sided
    realized harm, and blended criminal conduct trigger
    `ESCALATE_TO_HUMAN`.
-   Escalation overrides completeness: a fully detailed case still
    escalates when a risk signal is present.

### Ticket completeness check

-   `CREATE_TICKET` requires all three: an indicated relationship, two
    identifiable parties, and a specific dispute or event.
-   If any is missing, the rule falls through to
    `ASK_FOR_MORE_INFO`.

### Strict output handling

-   JSON-only output contract with a `High | Medium | Low` confidence
    label.
-   Regex extraction of the JSON object, tolerant of extra text or
    fences.
-   Validation of required keys, enum membership, and stage
    permission.
-   `temperature=0` for stable classification.

### Purpose-built test suites

-   A basic suite of six everyday messages.
-   **Adversarial cases** that each document the trap they set
    (disguised advice requests, verbal denial of intent, instruction
    injection, blended criminal content) and flag known limitations.
-   **Ambiguous cases** that record the expected action, the
    reasoning, and an explicit reason for rejecting each of the other
    four actions.

------------------------------------------------------------------------

## 3. Architecture

| Component | Responsibility |
|-----------|----------------|
| `Action` enum | Defines the five valid actions. |
| Allowed-action sets | `PART1_ALLOWED_ACTIONS` (3 actions) and `PART3_ALLOWED_ACTIONS` (all 5). |
| `Decision` | Frozen result object: action, confidence, reasoning. |
| System prompts | Part 1 and Part 3 rule prompts with priority ordering. |
| `call_model` | Sends system and user messages to the LLM and returns raw text. |
| `extract_json` | Finds and parses the JSON object in the raw response. |
| `validate_decision` | Checks keys, enum membership, and stage permission. |
| `classify_message` | Generic entry point reusable across prompts and action sets. |
| `AdversarialCase` | Test case with expected action, trap description, and known-limitation flag. |
| `AmbiguousCase` | Test case with expected action, reasoning, and reasons for each rejected alternative. |
| Test runners | Run the basic, adversarial, and ambiguous suites and print verdicts. |

### Rule priority (Part 3)

| Priority | Condition | Action |
|----------|-----------|--------|
| 1 | Asks for a legal opinion, validity judgment, clause explanation, or "what should I do" | `REJECT_REQUEST` |
| 2 | No signal of any formal agreement or relationship | `FLAG_OUT_OF_SCOPE` |
| 3 | Contract-related with a risk signal | `ESCALATE_TO_HUMAN` |
| 4 | Relationship, two parties, and a specific dispute all present | `CREATE_TICKET` |
| 4 (fallback) | Any of those three missing | `ASK_FOR_MORE_INFO` |

### High-level workflow

``` text
        User message
              |
              v
   Select prompt + allowed actions
        (Part 1 or Part 3)
              |
              v
        LLM (temperature 0)
              |
              v
     Extract JSON from output
              |
              v
      Validate decision
   (keys, enum, allowed at stage)
          /           \
      Valid          Invalid
        |               |
        v               v
   Decision object   ValueError reported
   (action, confidence,  by the test runner
    reasoning)
        |
        v
   Compared with the expected action
   in the test suites (PASS / FAIL /
   KNOWN LIMITATION)
```

------------------------------------------------------------------------

## 4. Technical Highlights

### Stage-aware validation

Validation takes the allowed set as a parameter. A well-formed answer is
still rejected if it names an action that is not available at the
current stage, which keeps each stage honest and lets the same code
serve every part.

### Ordered rules instead of a flat list

The prompt does not just list actions; it orders the rules and tells the
model to stop at the first match. This makes the intended precedence
explicit, for example that a request for an opinion wins over a fully
detailed dispute, and that a risk signal wins over completeness.

### Tests that document the trap

Each adversarial case states what it is trying to trick the system into
doing, such as a verbal denial of seeking advice or an embedded command
to open a ticket. Each ambiguous case explains why every other action is
wrong. This turns the test suite into a written specification of the
decision logic.

### Recorded results

The saved outputs show how the prompts behaved:

-   **Basic suite (Part 1):** advice-style messages were rejected, vague
    contract messages asked for more information, and the parking
    ticket was flagged as out of scope.
-   **Adversarial suite (Part 1 prompt):** 3 of 5 cases matched the
    expected action. Two cases, an informal debt agreement and a
    threat framed as blackmail, were rejected as advice requests
    instead of asking for more information.
-   **Ambiguous suite (Part 3 prompt):** 2 of 3 cases matched. The
    concealment case escalated and the legal-validity question was
    rejected, but the vague logistics dispute was turned into a ticket
    instead of asking for more information.

### Reusable entry point

`classify_message(message, system_prompt, allowed_actions)` takes the
prompt and the allowed set as arguments, so new stages or prompt
variants can be tested without rewriting the pipeline.

------------------------------------------------------------------------

## 5. Technology Stack

| Technology | Role |
|------------|------|
| Python | Core implementation language. |
| Google Colab | Notebook execution environment and secret storage. |
| `openai` client | OpenAI-compatible client used with the Hugging Face router. |
| Hugging Face router | Hosted inference endpoint for the model. |
| Qwen3-Coder-Next (via Novita) | Language model used for classification. |
| `enum` / `dataclasses` / `typing` | Actions, decisions, and test-case definitions. |
| `json` / `re` | Response extraction and parsing. |

------------------------------------------------------------------------

## 6. Running the Notebook

### Requirements

The notebook is designed for Google Colab and needs a Hugging Face API
key available as the `HF_API_KEY` environment variable.

Before running:

-   Open the notebook in Google Colab.
-   Add `HF_API_KEY` to Colab secrets and enable notebook access.
-   Make sure the `openai` package is available in the runtime.
-   Read the key from secrets or the environment only; do not paste it
    into a cell.

### Execution order

Run the notebook cells from top to bottom. The main stages are:

1.  Load the API key and model configuration.
2.  Define the actions, allowed sets, and result objects.
3.  Define the test-case structures.
4.  Define the Part 1 and Part 3 system prompts.
5.  Define the model call, JSON extraction, validation, and the generic
    classifier.
6.  Run the basic test suite.
7.  Define and run the adversarial suite.
8.  Define and run the ambiguous suite.

### Example interaction

A message such as:

``` text
Can you explain what 'force majeure' means in my contract?
```

is classified as `REJECT_REQUEST`, because it asks for a clause
explanation. A message such as:

``` text
I got a parking ticket and want to dispute it.
```

is classified as `FLAG_OUT_OF_SCOPE`, because it has no contract
signal. Exact results depend on the model's output.

------------------------------------------------------------------------

## 7. Configuration

Important settings are defined in a few cells so they can be reviewed
and adjusted without changing the structure.

Review these settings:

-   `MODEL_NAME`: the model and provider route.
-   `temperature` in `call_model` (0).
-   `SYSTEM_PROMPT` and `SYSTEM_PROMPT_PART3`: rule text and ordering.
-   `PART1_ALLOWED_ACTIONS` and `PART3_ALLOWED_ACTIONS`.
-   `TEST_MESSAGES`, `ADVERSARIAL_TEST_CASES`, and
    `AMBIGUOUS_TEST_CASES`.

Never hard-code API keys in a notebook or commit them to a public
repository. Use Colab secrets or environment variables, and revoke any
key that has been exposed.

------------------------------------------------------------------------

## 8. Security Considerations and Limitations

This notebook is an educational prototype, not a production-grade legal
intake system.

| Area | Current approach | Important limitation |
|------|------------------|----------------------|
| API key | The working code reads the key from the environment. | In the notebook as saved, an earlier cell also contains a key written directly in code; it must be revoked and removed. |
| Prompt injection | Prompts define rule order; one adversarial case tests an embedded command. | A prompt is not a security boundary; the model can still be steered by crafted input. |
| Classification quality | Rule-ordered prompts. | Saved results show misclassifications: two adversarial cases and one ambiguous case did not match the expected action. |
| Test verdicts | Expected action per case, printed as PASS or FAIL. | No assertions or aggregate score; one case marked as a known limitation actually returned a different action than the one recorded as its "limited" answer. |
| Output validation | Keys, enum, and stage permission are checked. | `confidence` and `reasoning` values are not validated, and the JSON regex is greedy. |
| Error handling | JSON and validation errors are reported per case. | API and network errors are not handled, and there is no repair pass or safe fallback action. |
| Confidence | Reported as High, Medium, or Low. | Not used for routing and not calibrated; wrong answers were returned with High confidence. |
| Coverage | Handcrafted cases, mostly Arabic and English single messages. | Small sample, no multi-turn input, and no measurement across many runs or models. |
| Notebook hygiene | Reusable functions and staged prompts. | Leftover cells (a duplicate client, an unused model variable, commented-out code) should be cleaned up. |

### Recommendations before production use

-   Revoke any exposed key and keep credentials in secrets only.
-   Add a retry or repair step and a safe fallback (for example,
    escalate to a human) when output is invalid or the call fails.
-   Use confidence in routing, for example escalating below a
    threshold.
-   Turn the suites into assertion-based tests with an aggregate score.
-   Expand the labeled set and run it repeatedly and across models.
-   Review the rule wording for the cases that failed, especially the
    boundary between "asks for advice" and "reports an event".
-   Treat all user text as untrusted input.

------------------------------------------------------------------------

## 9. Repository Structure

``` text
.
├── Legal_Contract_Agent.ipynb
└── README.md
```

Do not commit credentials. Remove any hard-coded key from the notebook
and its history before publishing.

------------------------------------------------------------------------

## 10. Possible Future Improvements

-   Add the missing middle stage that the staging comments refer to, so
    the progression from Part 1 to Part 3 is complete in the notebook.
-   Refine Rule 1 so event reports with no request for opinion are not
    rejected as advice requests.
-   Add a deterministic pre-check or a repair pass for invalid output.
-   Convert the suites to automated tests with assertions and a pass
    rate report.
-   Add a larger labeled set covering Arabic dialects and English, and
    track accuracy per action.
-   Compare different models and prompt variants on the same suites.
-   Connect the decision to downstream steps such as ticket storage and
    notification.
-   Add multi-turn handling for `ASK_FOR_MORE_INFO`.

------------------------------------------------------------------------

## 11. Key Takeaway

This project demonstrates how to build and test a constrained LLM
decision engine: the model classifies one message into one action, while
the surrounding code enforces the output contract and the allowed
actions, and the test suites document and probe the rules.

The most important design lesson is that **reliable agentic systems
require more than a capable model or a carefully written prompt: they
need ordered rules, validated outputs, and adversarial tests that show
where the system still fails.**
