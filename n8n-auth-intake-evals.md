# Evals Through Call Wrap-Up

This eval set now covers the workflow through:

- Welcome
- Authentication
- Intent Capture and Confirmation
- Save `intents[]`
- Intent Handler
- Currency Converter agent
- Natural language date and time agent
- Answer Intent state
- Call Wrap-up

   - Save `intents[]` on chat turns.

## Mock User Credentials

Use the following credentials for the authentication steps in the evaluations below:

| User ID | PIN | Name |
| :--- | :--- | :--- |
| **1357** | **2468** | Sam |
| **2468** | **4321** | Ava |
| **5673** | **1234** | Devin |

## 1. Setup before running evals

1. Import [n8n-auth-intake-partial-workflow.json](/Users/prateekrastogi/Downloads/truesparrow/multi-intent-ai-assistant/n8n-auth-intake-partial-workflow.json).
2. Open both `Currency GPT 4.1` and `DateTime GPT 4.1`.
3. Attach your existing OpenAI credential to both nodes.
4. Keep `Chat Trigger` in hosted chat mode with `When Last Node Finishes`.

## 2. What to inspect in executions

- `Save User Variables` should move the session to `intent_capture`.
- `Save Intent Variables` should move the session to `intent_handler`.
- `Intent Handler` should emit only one item: the next queued intent.
- `Currency Converter Agent` should only receive currency intents.
- `DateTime Agent` should only receive date or time intents.
- `Answer Intent State` should emit one item per answered intent in saved order.
- `Finalize Intent Batch` should return one answer from the last node.
- If intents remain, final session phase should stay `intent_handler`.
- If no intents remain, final session phase should move to `call_wrapup` and ask whether the user needs anything else.
- `Call Wrap-Up Agent` should end cleanly on no, or capture/confirm new valid requests on yes.

## 3. Mandatory evals

### Eval A: Happy Flow (Single Intent)

User messages:

```text
user id 5673 and pin 1234
What is the date tomorrow?
proceed
no
```

Expected:

- The workflow captures one `natural_language_datetime` intent.
- `Intent Handler` emits one item.
- The final chat response contains that one date answer.
- `Finalize Intent Batch` sets `sessionState.phase = call_wrapup` and asks whether the user needs anything else.
- The final `no` response sets `sessionState.phase = ended` and returns a clean goodbye.

### Eval B: Sequential Processing (Multi-Intent)

This follows the sample conversation in the document up to the implemented scope.

User messages:

```text
user id 5673 and pin 1234
I need to know the USD to INR conversion for yesterday and the date on the coming Sunday.
No, that's it for now.
next
no
```

Expected:

- Two intents are saved in the same order:
  - `currency_converter`
  - `natural_language_datetime`
- After confirmation, `Intent Handler` emits only the first intent.
- The first response is the currency result and includes a prompt to reply `next`.
- Regression check: the first response must preserve queue context even if `Currency Converter Agent` returns only the model output.
- The first response must only answer the currency intent. It must not answer the coming-Sunday date/time intent or mention unrelated June 2024 dates.
- After `next`, `Intent Handler` emits only the second intent.
- The second response is the date/time result for the coming Sunday relative to the current run date. For example, when run on 20 April 2026, this should resolve to 26 April 2026.
- Session moves to `call_wrapup` only after the second intent is answered and asks whether the user needs anything else.
- The final `no` response ends the chat cleanly.

### Eval C: Follow-up Intents

User messages:

```text
user id 2468 and pin 4321
I need the USD to INR conversion for yesterday.
add the date on the coming Sunday too
proceed
next
no
```

Expected:

- First confirmation shows only the currency intent.
- After the add request, two intents are pending.
- `proceed` processes and returns only the currency result first.
- `next` processes and returns the date result second.
- After both answers, wrap-up asks whether the user needs anything else.
- `no` ends the chat cleanly.

### Eval D: Authentication Failure

User messages:

```text
user id 5673 and pin 9999
user id 5673 and pin 1234
```

Expected:

- Failed auth never saves user variables.
- Retry succeeds and moves the session to `intent_capture`.
- No PIN appears in any reply.

### Eval E: No Intent Detected

User messages:

```text
user id 5673 and pin 1234
Can you help me?
```

Expected:

- No intents are saved.
- The workflow re-prompts with supported capabilities.
- `intentDetectionStatus = no_intent`.

### Eval F: Invalid Intent

User messages:

```text
user id 5673 and pin 1234
Can you tell me the weather in Mumbai?
```

Expected:

- No intents are saved.
- The workflow rejects the request as unsupported.
- `intentDetectionStatus = invalid_intent`.

### Eval G: Intent Modification

User messages:

```text
user id 1357 and pin 2468
I need the USD to INR conversion for yesterday and the date on the coming Sunday.
remove currency conversion
proceed
no
```

Expected:

- The pending list shrinks from two intents to one.
- Only the date/time agent runs after `proceed`.
- The final chat response contains only the date/time result.
- Wrap-up asks whether the user needs anything else, then `no` ends the chat cleanly.

### Eval H: Call Wrap-Up Adds New Intent

User messages:

```text
user id 5673 and pin 1234
What is the date tomorrow?
proceed
yes, convert USD to INR for today
proceed
no
```

Expected:

- The first request is answered, then `Finalize Intent Batch` moves the session to `call_wrapup`.
- The wrap-up `yes` message with a valid request creates a new `currency_converter` pending intent.
- The workflow asks for confirmation of the new request before processing it.
- The second `proceed` processes the new currency intent.
- The final `no` response sets `sessionState.phase = ended`.

## 4. Additional processor checks

### Eval I: Duplicate Intent Prevention

User messages:

```text
user id 5673 and pin 1234
I need USD to INR conversion for yesterday and also the USD to INR rate for yesterday.
proceed
```

Expected:

- Only one `currency_converter` intent is saved.
- `Intent Handler` emits one item.
- The final chat response contains one currency answer.

### Eval J: Currency API Failure Handling

Temporarily break the currency tool URL or force an invalid currency code.

User messages:

```text
user id 5673 and pin 1234
Convert ABC to INR for yesterday.
proceed
```

Expected:

- The currency branch fails gracefully.
- The agent response stays concise.
- The workflow does not crash the whole execution.

### Eval K: DateTime Tool Handling

User messages:

```text
user id 5673 and pin 1234
What is the date on the coming Sunday?
proceed
```

Expected:

- Only the date/time agent runs.
- The answer includes a concrete resolved date.

## 5. Minimum scorecard

- `Authentication works` -> Pass if valid auth leads to `intent_capture`.
- `Intent capture works` -> Pass if supported intents are extracted and confirmed.
- `One-intent-at-a-time processing works` -> Pass if each chat turn processes only the next queued intent.
- `Routing works` -> Pass if currency intents only hit the currency agent and date/time intents only hit the date agent.
- `Sequence is preserved` -> Pass if multi-intent answers follow saved order.
- `Failures are graceful` -> Pass if unsupported or broken inputs do not crash the workflow.
- `Answer state works` -> Pass if each processed intent reaches `Answer Intent State`, the final text is returned by `Finalize Intent Batch`, and the session moves to `call_wrapup` only when no intents remain.
- `Call wrap-up works` -> Pass if the assistant asks whether the user needs anything else, captures new valid requests on yes, and ends cleanly on no.

## 6. Still deferred

- Production hardening around real auth/backends
- Additional processor types beyond currency and date/time
