# Excerpt 01 — Request-Scoped Tools and the Write Guard

**Source:** `backend_service/app/agent/tools.py` (private repository)
**Lines shown:** 37–46, 49–68, 114–122, 145–175 of 225. Code is shown unmodified; gaps between blocks are omitted lines.

This excerpt shows why the language model cannot act as another user or turn a hallucinated machine ID into a reservation.

## The only argument the model can supply

The tool's input schema has a single field. Identity is not part of it.

```python
class ReserveMachineInput(BaseModel):
    machine_id: str = Field(
        default="",
        description=(
            "The exact Firestore document ID of the machine to reserve. "
            "This value MUST be taken verbatim from the output of get_available_machines. "
            "Do not guess, invent, or paraphrase this value. Leave it empty when the "
            "user has not selected a machine."
        ),
    )
```

## Deterministic guard: did the user actually name this machine?

The guard reads the user's original message, not the model's interpretation of it. It accepts the document ID, the display name, or a number pattern in English or Turkish (`_NAMED_MACHINE_PATTERNS`, defined at lines 14–20).

```python
def _message_selects_machine(message: str, machine: dict) -> bool:
    normalized_message = message.casefold()
    machine_id = str(machine.get("id", "")).strip()
    if machine_id and machine_id.casefold() in normalized_message:
        return True

    display_name = machine_display_name(machine)
    if display_name != "Washer" and display_name.casefold() in normalized_message:
        return True

    id_match = _MACHINE_ID_PATTERN.fullmatch(machine_id)
    if not id_match:
        return False

    machine_number = int(id_match.group(1))
    return any(
        int(match.group(1)) == machine_number
        for pattern in _NAMED_MACHINE_PATTERNS
        for match in pattern.finditer(message)
    )
```

## Tools built per request, with identity captured in a closure

`build_tools()` is called once for each `/chat` request. `user_uid`, `full_name`, `request_id` and `user_message` come from the verified token and the request, and are visible to the tool body but not to the model.

```python
def build_tools(
    firebase_service: FirebaseService,
    *,
    user_uid: str,
    full_name: str,
    request_id: str,
    user_message: str,
):
    """Build tools whose trusted identity and idempotency key are server-bound."""
```

*Lines 123–144 (the read-only `get_available_machines` tool) are omitted.*

```python
    @tool(args_schema=ReserveMachineInput)
    def reserve_machine(machine_id: str = "") -> str:
        """Immediately reserve the machine explicitly selected by the user.

        Call get_available_machines first. If the user did not select a specific
        machine, omit machine_id; this tool will safely request clarification.
        """
        machines = firebase_service.get_all_machines()
        normalized_id = machine_id.strip() if isinstance(machine_id, str) else ""
        selected_machine = next(
            (machine for machine in machines if machine.get("id") == normalized_id),
            None,
        )

        if not normalized_id:
            return _clarification_result(machines, "MACHINE_SELECTION_MISSING")

        if selected_machine is None:
            return _clarification_result(machines, "MACHINE_ID_INVALID_OR_UNAVAILABLE")

        if not _message_selects_machine(user_message, selected_machine):
            return _clarification_result(machines, "MACHINE_NOT_SELECTED_BY_USER")

        display_name = machine_display_name(selected_machine)
        try:
            result = firebase_service.reserve_machine(
                machine_id=normalized_id,
                user_uid=user_uid,
                full_name=full_name,
                idempotency_key=f"agent-{request_id}",
            )
```

*Lines 176–217 are omitted. They map 404/409 transaction errors to `RESERVATION_NOT_COMPLETED` or `RESERVATION_CLARIFICATION_REQUIRED`, and return `RESERVATION_SUCCESS` only when the transaction result carries the same `machine_id`.*

## What to notice

- The order of checks: missing ID → unknown ID → **not named by the user** → transaction. Only the last step writes.
- `user_uid=user_uid` and `full_name=full_name` come from the closure. Nothing the model returns can reach those parameters.
- The idempotency key `agent-{request_id}` makes repeated tool calls within one chat request replay instead of double-booking.

Related: [AI tool-calling workflow](../docs/ai-tool-calling.md)
