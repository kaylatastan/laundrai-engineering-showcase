# Excerpt 03 — Token Verification and Identity Binding

**Sources (private repository):**
- `backend_service/app/dependencies.py`, lines 27–59 of 59
- `backend_service/app/routers/chat.py`, lines 15–32 of 40

Code is shown unmodified.

## Verifying the Firebase ID token

Every protected route depends on `get_current_user`. Expired, invalid and otherwise unverifiable tokens each produce a 401 with a fixed message. The underlying exception is never returned to the client; unexpected verification failures are logged server-side.

```python
def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(_bearer_scheme)],
    firebase_service: Annotated[FirebaseService, Depends(get_firebase_service)],
) -> dict:
    """Verify the Firebase ID token from the Authorization: Bearer header.

    Returns the decoded token claims dict, which includes the verified ``uid``.
    Raises HTTP 401 on invalid or expired tokens.
    """
    # Ensure Firebase app is initialised (singleton via get_firebase_service)
    _ = firebase_service
    try:
        decoded = firebase_admin.auth.verify_id_token(credentials.credentials)
        return decoded
    except firebase_admin.auth.ExpiredIdTokenError as exc:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Firebase ID token has expired.",
            headers={"WWW-Authenticate": "Bearer"},
        ) from exc
    except firebase_admin.auth.InvalidIdTokenError as exc:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid Firebase ID token.",
            headers={"WWW-Authenticate": "Bearer"},
        ) from exc
    except Exception as exc:
        logger.exception("Firebase ID token verification failed")
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not verify Firebase ID token.",
            headers={"WWW-Authenticate": "Bearer"},
        ) from exc
```

## Binding the verified identity into the agent

The chat route takes the UID only from verified claims, generates a fresh `request_id`, and passes both to the agent service, which builds the request-scoped tools shown in [excerpt 01](01-request-scoped-tools.md). The request body (`ChatRequest`) is configured with `extra="forbid"` and contains only `message`, so a client cannot send identity fields at all.

```python
@router.post("/chat", response_model=ChatResponse)
async def chat(
    payload: ChatRequest,
    current_user: Annotated[dict, Depends(get_current_user)],
    agent_service: Annotated[LaundryAgentService, Depends(get_agent_service)],
) -> ChatResponse:
    request_id = str(uuid4())
    try:
        user_uid = current_user["uid"]
        email = current_user.get("email") or ""
        full_name = current_user.get("name") or email.split("@")[0] or "Student"
        response = await agent_service.run(
            message=payload.message,
            user_uid=user_uid,
            full_name=full_name,
            request_id=request_id,
        )
        return ChatResponse(response=response)
```

*Lines 33–40 re-raise `HTTPException` and convert any other failure into a logged, generic HTTP 500: "The assistant is temporarily unavailable."*

## What to notice

- The UID flows token → claims → route → agent service → tool closure. At no point is it read from the request body or from model output.
- The reservation routes follow the same rule: `ReservationRequest.user_uid` exists for client compatibility, but the route uses `current_user["uid"]`.

Related: [Architecture — authentication and UID binding](../docs/architecture.md#authentication-and-uid-binding)
