# Evals for Partial Workflow Through Save Intents

This eval set is scoped to the workflow slice that is implemented now:

- Welcome
- Authentication with mock user data
- Save `userId` and `name`
- Intent Capture and Confirmation
- Save `intents[]`

The remaining stages from the document are still deferred:

- Intent Handler
- Intent Processors
- Call Wrap-up

## 1. Import and setup

1. Import [n8n-auth-intake-partial-workflow.json](/Users/prateekrastogi/Downloads/truesparrow/multi-intent-ai-assistant/n8n-auth-intake-partial-workflow.json).
2. Open `Chat Trigger`.
3. Keep `Response Mode` set to `When Last Node Finishes`.
4. Use the hosted chat URL if you want to see the configured initial message.

## 2. Mock users included in the workflow

- `5673` / `1234` -> `Devin`
- `2468` / `4321` -> `Ava`
- `1357` / `2468` -> `Sam`

## 3. What to verify after each run

Check the last node output and the relevant intermediate nodes:

- `Save User Variables` shows `sessionState.phase = intent_capture` after successful authentication.
- `Summarize And Confirm Intents` returns a concise confirmation summary for valid detected intents.
- `Confirmation Agent` supports `proceed`, add, and remove flows.
- `Save Intent Variables` shows `savedIntents` and `sessionState.phase = intent_handler` after confirmation.
- Replies never expose the raw PIN.
- Duplicate intents are not saved twice.

## 4. Mandatory eval coverage for the current scope

### Eval A: Happy Flow (Single Intent)

User messages:

```text
user id 5673 and pin 1234
What is the date tomorrow?
proceed
```

Expected:

- Auth response is `Hi Devin, how can I help you today?`
- Intent summary confirms one date or time intent.
- `Save Intent Variables` stores one intent of type `natural_language_datetime`.
- Final phase becomes `intent_handler`.

### Eval B: Sequential Processing (Multi-Intent Intake)

This mirrors the sample conversation in the document up to the save step.

User messages:

```text
user id 5673 and pin 1234
I need to know the USD to INR conversion for yesterday and the date on the coming Sunday.
No, that's it for now.
```

Expected:

- The summary mentions both requests in the same order as the user asked them.
- `savedIntents` contains exactly two intents.
- The first intent type is `currency_converter`.
- The second intent type is `natural_language_datetime`.
- Final phase becomes `intent_handler`.

### Eval C: Follow-up Intents

User messages:

```text
user id 2468 and pin 4321
I need the USD to INR conversion for yesterday.
add the date on the coming Sunday too
proceed
```

Expected:

- First summary contains only the currency conversion intent.
- After the add message, the confirmation summary contains both intents.
- `savedIntents` contains two unique intents.
- No duplicate currency intent is introduced.

### Eval D: Authentication Failure

User messages:

```text
user id 5673 and pin 9999
user id 5673 and pin 1234
```

Expected:

- Failed login returns a generic auth failure response.
- No user variables are saved on the failed attempt.
- The retry succeeds and moves the session to `intent_capture`.

### Eval E: No Intent Detected

User messages:

```text
user id 5673 and pin 1234
Can you help me?
```

Expected:

- The workflow does not save intents.
- It re-prompts with the supported capabilities.
- `intentDetectionStatus` is `no_intent`.

### Eval F: Invalid Intent

User messages:

```text
user id 5673 and pin 1234
Can you tell me the weather in Mumbai?
```

Expected:

- The workflow does not save intents.
- It explains that only currency conversion and natural language date or time are supported.
- `intentDetectionStatus` is `invalid_intent`.

### Eval G: Intent Modification

User messages:

```text
user id 1357 and pin 2468
I need the USD to INR conversion for yesterday and the date on the coming Sunday.
remove currency conversion
proceed
```

Expected:

- The first summary contains two intents.
- After removal, the updated summary contains only the date or time intent.
- `savedIntents` contains exactly one intent of type `natural_language_datetime`.

## 5. Additional edge checks

### Eval H: Duplicate Intent Prevention

User messages:

```text
user id 5673 and pin 1234
I need USD to INR conversion for yesterday and also the USD to INR rate for yesterday.
proceed
```

Expected:

- The summary includes only one currency conversion intent.
- `savedIntents` contains exactly one `currency_converter` intent.

### Eval I: Remove All Intents

User messages:

```text
user id 5673 and pin 1234
What is the date tomorrow?
remove date
```

Expected:

- Pending intents become empty.
- The workflow moves back to `intent_capture`.
- The response asks the user to share a supported request again.

## 6. Minimum scorecard for this milestone

- `Auth works` -> Pass if valid credentials move the session to `intent_capture`.
- `No PIN leakage` -> Pass if no response contains the raw PIN.
- `No intent re-prompt works` -> Pass if generic requests are redirected to supported capabilities.
- `Invalid intent handling works` -> Pass if unsupported requests are rejected cleanly.
- `Multi-intent capture works` -> Pass if the workflow stores both supported intents from one message.
- `Intent modification works` -> Pass if add and remove update the pending list correctly.
- `Save intents works` -> Pass if `savedIntents` appears and phase becomes `intent_handler`.

## 7. Deferred evals

These still need to be added after the next workflow extension:

- Sequential processing of one saved intent at a time by the Intent Handler
- Actual processor outputs for currency conversion
- Actual processor outputs for natural language date and time
- Call Wrap-up and post-processor follow-up requests
