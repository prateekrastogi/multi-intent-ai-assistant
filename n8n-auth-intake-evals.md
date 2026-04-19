# Partial Evals for Welcome + Authentication

This eval set is intentionally scoped only to the part implemented right now:

- Welcome
- Authentication with mock user data
- Save `userId` and `name`

The later must-have evals from the document are deferred until we add intent capture and intent processing.

## 1. Import and basic setup

1. Import [n8n-auth-intake-partial-workflow.json](/Users/prateekrastogi/Downloads/truesparrow/n8n-auth-intake-partial-workflow.json).
2. Open `Chat Trigger`.
3. Keep it in test mode first.

## 2. Mock users included in the workflow

Use these built-in test accounts:

- `5673` / `1234` -> `Devin`
- `2468` / `4321` -> `Ava`
- `1357` / `2468` -> `Sam`

## 3. What to verify after each run

Check the final node output and the execution data:

- The reply is short and does not expose the PIN.
- On success, the workflow responds with `Hi <Name>, how can I help you today?`
- In `Save User Variables`, `savedUser.userId` and `savedUser.name` are present.
- In failures, `savedUser` is absent and the workflow re-prompts for credentials.
- Session state in workflow static data moves to `intent_capture` only after a valid login.

## 4. Eval cases to run now

### Eval A: Happy Flow (Authentication Success)

User messages:

```text
user id 5673 and pin 1234
```

Expected:

- Response is `Hi Devin, how can I help you today?`
- Saved variables contain `5673` and `Devin`.
- PIN is never echoed back.

### Eval B: Authentication Failure

User messages:

```text
user id 5673 and pin 9999
user id 5673 and pin 1234
```

Expected:

- Failed login returns a generic auth failure message.
- Correct retry succeeds.
- No user details beyond the successful greeting are revealed before success.

### Eval C: Missing Input Format

User messages:

```text
my pin is 1234
user id 5673
user id 5673 and pin 1234
```

Expected:

- Workflow asks for both values in one message until both are present.
- It does not authenticate on partial input.
- Final retry succeeds and saves the user variables.

### Eval D: Security Check

User messages:

```text
user id 2468 and pin 4321
```

Expected:

- Response is `Hi Ava, how can I help you today?`
- Output must not contain `4321`.
- Saved variables contain only the user ID and name.

## 5. Minimum scorecard for this partial milestone

Use this pass/fail grid before moving to intent capture:

- `Welcome prompt shown` -> Pass if the user is asked for ID and PIN.
- `Credential parsing works` -> Pass if the workflow extracts both values from natural language.
- `Invalid auth rejected` -> Pass if wrong credentials do not save user variables.
- `Valid auth saved` -> Pass if `userId` and `name` are stored.
- `PIN never exposed` -> Pass if no response contains the raw PIN.

## 6. Deferred evals from the document

These are not part of this partial build yet:

- Sequential Processing (Multi-Intent)
- Follow-up intents
- No Intent Detected
- Invalid Intent
- Intent Modification
