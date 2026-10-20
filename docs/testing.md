# Testing

## Results

| Suite | Framework | Tests | Last local run (October 2026) |
| --- | --- | --- | --- |
| Flutter client | `flutter_test` | **118** in 11 tracked test files | All passed; `flutter analyze` reported no issues |
| FastAPI backend | Python `unittest`, `unittest.mock` | **45** in 4 test files | All passed |

There is no CI pipeline. These numbers come from local runs of the private repository.

A note on the Flutter count: running `flutter test` in the working copy reports 119 tests. The extra test is the default Flutter template test inside an untracked scaffold directory (`test/laundrysystem/`) that is excluded by `.gitignore`. The tracked suite contains 118 tests.

The backend tests mock Firestore and never call the LLM, so they run without credentials or network access.

## Inventory

### Backend (45)

| File | Tests | Focus |
| --- | --- | --- |
| `tests/test_agent_response_contract.py` | 21 | Display names, tool schema, identity binding, write guard, success/failure tokens, prompt invariants, bare-confirmation interception |
| `tests/test_cancel_reservation.py` | 15 | Cancellation authorization, each rejection with a zero-write assertion |
| `tests/test_reservation_service.py` | 6 | 404 handling, 2-reservation ceiling from machine documents, counter reconciliation, idempotent replay, key reuse |
| `tests/test_chat_route.py` | 3 | Unauthenticated rejection, verified identity reaching the agent, `ChatRequest` rejecting identity fields |

### Flutter (118)

| File | Tests |
| --- | --- |
| `test/machines_screen_test.dart` | 21 |
| `test/auth_screen_test.dart` | 18 |
| `test/schedule_screen_test.dart` | 18 |
| `test/models_test.dart` | 17 |
| `test/history_screen_test.dart` | 15 |
| `test/assistant_chat_test.dart` | 12 |
| `test/widget_test.dart` | 5 |
| `test/backend_url_test.dart` | 4 |
| `test/user_name_sync_test.dart` | 4 |
| `test/profile_screen_test.dart` | 3 |
| `test/chat_service_test.dart` | 1 |

### Not covered

- Real Firestore transaction semantics (no emulator tests). Transaction logic is tested with mocks that record reads and writes.
- HTTP-level tests of the reservation routes.
- The Firebase-connected parts of the client.

## Selected Tests

All test code below is copied verbatim from the private repository. Paths are relative to its root.

### 1. The model cannot see or supply identity

`backend_service/tests/test_agent_response_contract.py`, lines 92–119. The helper `_build_tools` (lines 46–53) builds the tools with `user_uid="verified-user"` and a fake Firestore service that records every reservation call.

```python
    def test_llm_visible_schema_contains_machine_id_only(self):
        reserve_tool = _build_tools(_FakeFirebaseService([self.machine]))[1]

        self.assertEqual(set(reserve_tool.args), {"machine_id"})
        self.assertNotIn("user_uid", str(reserve_tool.args))
        self.assertNotIn("full_name", str(reserve_tool.args))

    def test_verified_identity_is_bound_server_side(self):
        firebase = _FakeFirebaseService([self.machine])
        reserve_tool = _build_tools(firebase)[1]

        reserve_tool.invoke({"machine_id": "machine_010"})

        self.assertEqual(firebase.reserve_calls[0]["user_uid"], "verified-user")
        self.assertEqual(firebase.reserve_calls[0]["full_name"], "Verified User")

    def test_vague_request_proposes_machine_without_writing(self):
        firebase = _FakeFirebaseService([self.machine])
        availability_tool, reserve_tool = _build_tools(
            firebase, message="can you reserve for me"
        )

        self.assertIn("DISPLAY_NAME: Washer 10", availability_tool.invoke({}))
        result = reserve_tool.invoke({"machine_id": "machine_010"})

        self.assertIn("RESERVATION_CLARIFICATION_REQUIRED", result)
        self.assertIn("DISPLAY_NAME: Washer 10", result)
        self.assertEqual(firebase.reserve_calls, [])
```

The third test is the write guard: the model picked a valid, idle machine, but the user's message ("can you reserve for me") never named it, so no reservation call reaches the service.

### 2. Rejected cancellations perform zero writes

`backend_service/tests/test_cancel_reservation.py`. The shared assertion (lines 88–99) checks that a rejected request raised the expected status **and** that the transaction issued no `update`, `set`, `delete` or `create`:

```python
    def assertNoWrites(self, txn):
        self.assertFalse(txn.update.called)
        self.assertFalse(txn.set.called)
        self.assertFalse(txn.delete.called)
        self.assertFalse(txn.create.called)

    def assertRejected(self, service, txn, status_code, **kwargs):
        with self.assertRaises(HTTPException) as raised:
            _cancel(service, **kwargs)
        self.assertEqual(raised.exception.status_code, status_code)
        self.assertNoWrites(txn)
        return raised.exception
```

Two of the fifteen tests that use it (lines 143–157). Both cover the client-supplied reservation ID. They also assert that the ID is never even used to look up a document:

```python
    def test_cannot_cancel_another_users_reservation_via_own_machine(self):
        # u1 owns m1 (res-1) and tries to target u2's reservation document.
        service, txn, refs = _service(
            machine=_machine(),
            reservation=_reservation(userId="u2", machineId="m9"),
            reservation_id="res-2",
        )
        self.assertRejected(service, txn, 409, reservation_id="res-2")
        # The client supplied ID must never be fetched or written.
        refs["reservation_collection"].document.assert_not_called()

    def test_client_reservation_id_mismatch_is_rejected_without_writes(self):
        service, txn, refs = _service(machine=_machine(), reservation=_reservation())
        self.assertRejected(service, txn, 409, reservation_id="res-other")
        refs["reservation_collection"].document.assert_not_called()
```

### 3. Idempotent retries and key reuse

`backend_service/tests/test_reservation_service.py`, lines 88–109. The first test simulates an existing idempotency record for the same machine; the machine is already `reserved`, yet the call returns the stored result because the idempotency check runs first, and the transaction issues no `set`. The second reuses the key for a different machine.

```python
    def test_retry_returns_original_result_without_creating_a_second_reservation(self):
        original = {
            "reservation_id": "original-reservation",
            "machine_id": "m1",
            "user_uid": "u1",
            "user_name": "Alice",
            "status": "active",
            "reserved_at": "2026-01-01T00:00:00+00:00",
            "idempotent_replay": False,
        }
        service, _ = _service(machine={"status": "reserved"}, request={"machineId": "m1", "result": original})
        result = self._call(service)
        self.assertEqual(result["reservation_id"], "original-reservation")
        self.assertTrue(result["idempotent_replay"])
        self.assertFalse(service._db.transaction.return_value.set.called)

    def test_idempotency_key_cannot_be_reused_for_another_machine(self):
        service, _ = _service(machine={"status": "idle"}, request={"machineId": "m2", "result": {}})
        with self.assertRaises(HTTPException) as raised:
            self._call(service)
        self.assertEqual(raised.exception.status_code, 409)
        self.assertEqual(raised.exception.detail, "IDEMPOTENCY_KEY_REUSED")
```

### 4. Fail-closed client configuration

`test/backend_url_test.dart`, lines 1–37. The Flutter client refuses to call the backend without a valid `BACKEND_URL`, and its error message does not echo the invalid value.

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:smart_laundry_system/services/backend_url.dart';

void main() {
  group('BackendUrl.parse', () {
    test('rejects a missing URL', () {
      expect(() => BackendUrl.parse(''),
          throwsA(isA<BackendUrlConfigurationError>()));
    });

    test('rejects invalid or relative URLs', () {
      expect(() => BackendUrl.parse('laundry-api'),
          throwsA(isA<BackendUrlConfigurationError>()));
      expect(() => BackendUrl.parse('ftp://example.com'),
          throwsA(isA<BackendUrlConfigurationError>()));
    });

    test('does not echo an invalid configuration value', () {
      const invalidValue = 'invalid-sensitive-host';
      expect(
        () => BackendUrl.parse(invalidValue),
        throwsA(
          isA<BackendUrlConfigurationError>().having(
            (error) => error.toString(),
            'message',
            isNot(contains(invalidValue)),
          ),
        ),
      );
    });

    test('accepts absolute http(s) URLs and removes a trailing slash', () {
      expect(BackendUrl.parse('https://api.example.com/'),
          'https://api.example.com');
    });
  });
}
```

### 5. One source of truth for the user's name

`test/user_name_sync_test.dart`, lines 31–46. A profile save updates the sidebar without a reload.

```dart
  testWidgets('sidebar follows a saved profile name without a reload',
      (tester) async {
    useDesktop(tester);
    final notifier = UserNameNotifier()..value = 'Ada Lovelace';
    addTearDown(notifier.dispose);

    await tester.pumpWidget(shell(notifier: notifier));
    expect(find.text('Ada'), findsOneWidget);
    expect(find.text('Fallback'), findsNothing);

    // A profile save emits the new fullName from users/{uid}.
    notifier.value = 'Grace Hopper';
    await tester.pump();
    expect(find.text('Grace'), findsOneWidget);
    expect(find.text('Ada'), findsNothing);
  });
```

## How the suites are run (private repository)

```bash
# Flutter
flutter analyze
flutter test

# Backend (from backend_service/, virtualenv active)
python -m unittest discover -s tests -t . -v
```
