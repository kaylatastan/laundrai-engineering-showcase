# Excerpt 02 — Reservation Transaction with Idempotency

**Source:** `backend_service/app/services/firebase_service.py`, method `FirebaseService.reserve_machine` (private repository)
**Lines shown:** 110–118, 121–171, 203–214 of 363. Code is copied verbatim; gaps between blocks are omitted lines.

The same method serves the REST endpoint `POST /reservations/reserve` and the assistant's `reserve_machine` tool. `_MAX_ACTIVE_RESERVATIONS` is defined as `2` at the top of the module.

## Hashed, UID-scoped idempotency record

```python
        # A deterministic, per-user document makes retries safe without exposing
        # raw client input in a Firestore document path.
        request_digest = hashlib.sha256(
            f"{user_uid}:{idempotency_key}".encode("utf-8")
        ).hexdigest()
        request_ref = (
            self.db.collection(self.settings.reservation_requests_collection)
            .document(request_digest)
        )
```

## Reads first, then checks

Firestore transactions require every read to happen before any write.

```python
        transaction = self.db.transaction()

        @firestore.transactional
        def _run(txn):
            # All reads must precede writes in a Firestore transaction
            request_snap = request_ref.get(transaction=txn)
            machine_snap = machine_ref.get(transaction=txn)
            # Machine documents are the source of truth during migration: this
            # includes reservations created before users.activeReservationCount
            # existed. Reading the query in this transaction also serializes
            # concurrent reservations for the same user through user_ref below.
            active_machine_snaps = self.db.collection(
                self.settings.machines_collection
            ).where(
                filter=FieldFilter("reservedByUid", "==", user_uid)
            ).get(transaction=txn)

            if request_snap.exists:
                request_data = request_snap.to_dict() or {}
                if request_data.get("machineId") != machine_id:
                    raise HTTPException(
                        status_code=status.HTTP_409_CONFLICT,
                        detail="IDEMPOTENCY_KEY_REUSED",
                    )
                result = dict(request_data["result"])
                result["idempotent_replay"] = True
                return result

            if not machine_snap.exists:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail=f"Machine '{machine_id}' was not found.",
                )

            machine_data = machine_snap.to_dict() or {}
            if machine_data.get("status") != "idle":
                raise HTTPException(
                    status_code=status.HTTP_409_CONFLICT,
                    detail=f"Machine '{machine_id}' is not idle (status: {machine_data.get('status')}).",
                )

            active_count = sum(
                1
                for snap in active_machine_snaps
                if (snap.to_dict() or {}).get("status") in {"reserved", "running"}
            )
            if active_count >= _MAX_ACTIVE_RESERVATIONS:
                raise HTTPException(
                    status_code=status.HTTP_409_CONFLICT,
                    detail="MAX_RESERVATIONS_EXCEEDED",
                )
```

*Lines 172–202 are omitted. They write the new `reservations` document with status `active`, update the machine to `reserved` with `reservedBy`, `reservedByUid`, `activeReservationId` and `startTime`, and build the `result` dictionary.*

## Reconciled counter and idempotency record, in the same transaction

```python
            # Reconcile rather than increment. This remains correct when the
            # profile or its counter is missing/stale, and makes user_ref the
            # transaction contention point for concurrent reserve requests.
            txn.set(user_ref, {"uid": user_uid, "activeReservationCount": active_count + 1}, merge=True)
            txn.set(request_ref, {
                "userUid": user_uid,
                "machineId": machine_id,
                "reservationId": reservation_id,
                "result": result,
                "createdAt": now,
            })
            return result
```

*Lines 215–224 run `_run(transaction)`, re-raise `HTTPException`, and wrap any other error in an HTTP 500.*

## What to notice

- A retry with the same key and machine returns the stored result with `idempotent_replay = True`; the route maps that to HTTP 200 instead of 201.
- The active-reservation limit is counted from machine documents, so it stays correct even if the cached `activeReservationCount` is missing or stale. The counter is then set to the computed value rather than incremented.
- All four writes (reservation, machine, user counter, idempotency record) commit together or not at all.

Related: [Reservation transactions and security design](../docs/reservations-and-security.md)
