# Day 1 — IAM: The Simple Version

If the detailed docs feel too complex, read ONLY this first.
It gives you the working mental model. Then go back to
01a when you want the depth — it will make sense on the
second pass.

---

## The building security analogy (AWS)

Your cloud account is a building. Every API request is someone
asking to enter a room.

| IAM thing | In the building |
|---|---|
| User | A person with a permanent badge (avoid these) |
| Role | A temporary visitor pass, borrowed for an hour |
| Identity policy | What is printed ON YOUR badge: "may enter rooms 1, 2" |
| Resource policy | A note taped ON A DOOR: "Company B people may enter" |
| Permission boundary | Building owner's rule: "no badge ever opens the server room" |
| SCP | "This floor doesn't exist for this employee" — even root can't pass |
| Cross-account | Visitor from another company needs TWO yeses: their company lets them go, your building lets them in |

### How security decides (the whole evaluation logic)

1. On the **banned list**? → out. (explicit deny always wins)
2. **Floor blocked** by the company? → out. (SCP)
3. **Door note** OR **badge** allows it? Either one is enough. → in
4. **Building rule** (boundary) also allows it? It can only veto. → in
5. Nobody said yes? → out. (default deny)

Steps 3's two sources are OR.
Step 4 is an AND — a second key that can only block, never open.

### The one S3 trap

- Reading a FILE needs the resource `my-bucket/*` (object actions)
- Listing the BUCKET needs `my-bucket` (bucket action)
- A read-only policy needs BOTH lines, or you get a confusing
  AccessDenied.

---

## GCP: two rules, that's all

```
Company (Organisation)
  └── Department (Folder)
        └── Project  ← one box per team / per app
```

**Rule 1 — grants flow down and can never be subtracted.**
Give someone Editor at the folder, and you cannot block them
from one project under it. There is no banned list in GCP.
So: grant small, grant low, never grant at the top "because
it was easier."

**Rule 2 — machines get their own identity (service account).**
- Attach it to the machine → good (no password anywhere)
- Borrow it for an hour (impersonation) → good
- Download its key file → NEVER (never expires; a leaked file
  IS the machine's identity)

---

## The 30-second recall

```
AWS:  banned list wins → floor (SCP) → door OR badge →
      ceiling (boundary) vetoes → default deny
      S3: two resource lines
      Cross-account: both sides must say yes

GCP: grants flow down, can't be subtracted
     grant small and low
     service accounts: attach or impersonate, never keys
```

Once this sticks, the "complex" docs are just vocabulary
hanging off these two pictures.
