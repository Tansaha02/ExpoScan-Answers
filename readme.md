# ExpoScan – Pre-Screening

**Candidate:** Tanmoy Saha
**Experience:** 2 years
**Stack:** Angular, React, Python, FastAPI, Spring Boot, AWS
**Date Submitted:**20th June 2026

---

## Question 1 – The Build

> Tell us about one production system you have personally owned end to end in the last two years.
> - What was the problem it solved?
> - What did you build and what were the key technical decisions?
> - Was it multi-tenant? If yes, how did you handle data isolation?
> - What broke in production and how did you fix it?

At TCS, I worked on a release management platform for Sony Pictures. The system managed both physical and digital content releases across multiple territories globally.

The core problem was that release data lived in silos. Different regional teams had their own processes, and a missed configuration could delay a title going live on a streaming platform or hold up physical distribution in an entire region.

What I built:

- An event-driven pipeline using AWS SQS and SNS to handle territory-specific release flows independently, so a failure in one region would not block others
- A React frontend with role-based access — regional coordinators saw only their territory data, global admins had full visibility
- Separate Spring Boot services for physical and digital channels, since the validation rules were fundamentally different for each
- S3 with a metadata layer for asset versioning, since release packages go through multiple revisions before they are locked

On isolation — this was not multi-tenant in the SaaS sense, but it was region-scoped. JWT tokens carried territory claims, and access was enforced at the API layer. A coordinator logged in for India had no access to UK release data, by design.

What broke — during a high-profile title launch, our SNS fan-out to digital distributors started dropping messages silently. No errors in the logs, just missing delivery confirmations downstream. After investigating, we found that Lambda concurrency limits were being hit under load, and messages were expiring in the queue before they could be processed.

The immediate fix was adding DLQs so nothing was lost. We raised concurrency limits and changed the confirmation model so that silence was treated as failure rather than success. Going forward every distributor had to send an explicit acknowledgement.

---

## Question 2 – The Architecture

> You are migrating ExpoScan - a single-tenant FastAPI + MongoDB PWA - to a multi-tenant SaaS product. Multiple clients will use the same deployment. Their data must be completely isolated.
> - Pick one data isolation approach and justify it for this specific context — a trade show lead capture app expecting 50 to 200 tenants at launch.
> - Write the FastAPI middleware or dependency that enforces tenant isolation on every request.
> - What is the single biggest risk in this migration and how do you mitigate it?

I would go with a shared collection and a `tenant_id` field on every document.

For a lead capture app with 50 to 200 tenants, running a separate database per tenant creates operational overhead that is not justified. You end up managing hundreds of connection pools, separate index configurations, and independent backup policies for what is essentially a simple schema. The complexity cost outweighs any isolation benefit at this scale.

Shared collection is the right call here, provided isolation is enforced centrally — not left to individual developers on individual routes.

**Middleware:**

```python
from fastapi import Request, HTTPException, Depends
from jose import jwt, JWTError

SECRET_KEY = "your-secret"
ALGORITHM = "HS256"

async def get_tenant(request: Request) -> str:
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    if not token:
        raise HTTPException(status_code=401, detail="Missing token")
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        tenant_id = payload.get("tenant_id")
        if not tenant_id:
            raise HTTPException(status_code=403, detail="No tenant claim")
        request.state.tenant_id = tenant_id
        return tenant_id
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Applied on every route that touches data
@app.get("/leads")
async def get_leads(tenant_id: str = Depends(get_tenant), db=Depends(get_db)):
    return await db.leads.find({"tenant_id": tenant_id}).to_list(100)
```

The biggest risk is a developer writing a query without the `tenant_id` filter, which would silently return data across tenants.

The mitigation is to wrap the MongoDB collection in a `TenantCollection` class that injects `tenant_id` into every query automatically. Route handlers never get access to the raw collection. One class owns isolation — it cannot be forgotten or bypassed by individual developers.

---

## Question 3 – The Debugging

> A FastAPI endpoint that has been running fine for three months suddenly starts returning 500 errors in production. The logs show a MongoDB timeout. No schema changes were made. No code was deployed. Traffic is normal.
> - What are the first three things you check and in what order?
> - What commands or tools do you actually run?
> - What is the most likely cause and how do you confirm it?

Nothing changed on the application side, so I start at the database layer, not the app logs.

**Step 1 — MongoDB server metrics**

I open Atlas Performance Advisor and run `db.currentOp()` to see what is happening live — active connections, query execution times, and whether any collection scans have spiked recently. This gives me a picture of the DB health before I look at anything else.

**Step 2 — Slow query profiler**

```js
db.setProfilingLevel(1, { slowms: 100 })
```

I look for queries that recently started doing full collection scans. Data volume alone can push a previously fast query into a COLLSCAN without any code or schema change.

**Step 3 — Connection pool**

```js
db.serverStatus().connections
```

A saturated connection pool looks identical to a database timeout in application logs. I rule this out before concluding it is a query issue.

The most likely cause is that the leads collection has grown to a point where a low-selectivity query now scans the entire collection and times out.

To confirm:

```js
db.leads.find({ status: "new" }).explain("executionStats")
```

If this shows `COLLSCAN` with a high `docsExamined` count, that is the issue. Add a compound index on the filtered fields and the problem is resolved.

---

## Question 4 – The Judgment Call

> You join as the sole engineer on a live product. In your first week you find: (1) no indexes on the leads collection which has 80,000 documents and is growing, (2) the Idempotency-Key check is not atomic - there is a race condition, (3) created_at timestamps are being set by the client, not the server.
> - Which do you fix first and why?
> - Write the fix for the one you consider most critical.
> - How do you communicate these findings to a non-technical founder without causing panic?

**Priority: Race condition first, then indexes, then timestamps.**

The idempotency race condition is the only issue causing silent data corruption in production right now. If two requests hit simultaneously with the same key, both can pass the check before either writes — resulting in duplicate leads. That means reps follow up on the same person twice, lead counts are inflated, and conversion data is unreliable. This is actively happening.

Missing indexes slow queries but return correct data. Client-set timestamps are poor practice but have not broken anything yet. Neither is urgent compared to the race condition.

**The fix — atomic upsert using `find_one_and_update`:**

```python
from pymongo import ReturnDocument
from datetime import datetime

async def create_lead(lead: LeadModel, db=Depends(get_db), tenant_id=Depends(get_tenant)):
    result = await db.leads.find_one_and_update(
        {
            "idempotency_key": lead.idempotency_key,
            "tenant_id": tenant_id
        },
        {
            "$setOnInsert": {
                **lead.dict(),
                "tenant_id": tenant_id,
                "created_at": datetime.utcnow()
            }
        },
        upsert=True,
        return_document=ReturnDocument.AFTER
    )
    return result
```

The `$setOnInsert` with `upsert=True` is atomic at the MongoDB level. If the key already exists, nothing is written. I also moved `created_at` to server-side here, which resolves the third issue at no extra cost.

A unique index as a hard safety net:

```python
await db.leads.create_index(
    [("tenant_id", 1), ("idempotency_key", 1)],
    unique=True
)
```

This ensures the database itself rejects duplicates even if application logic is somehow bypassed.

**Communicating to the founder:**

"I reviewed the codebase this week and found three things worth flagging. One needs attention now — in certain situations, the same lead can be recorded twice if two requests arrive at the same moment. This could cause your team to follow up with the same person more than once, and your lead numbers may be slightly off. I can fix this today with no downtime required. The other two are performance and data quality improvements I will take care of next week. None of these require any changes visible to your users."

---

*All code is my own. Happy to walk through any part of this on a call.*
