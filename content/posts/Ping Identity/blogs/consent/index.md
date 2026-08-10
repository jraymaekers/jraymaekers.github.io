---
title: Building an Audit-Proof Consent Engine in PingOne AIC/Ping AIS
date: 2026-08-07
subject: Blogs
topic: Consent
weight: 1
tags:
  - demo
  - poc
  - consent
  - AIC
summary: The WYSIWYA Pattern.
---
# Building an Audit-Proof Consent Engine: The WYSIWYA Pattern

Compliance auditors ask one question that most IAM implementations cannot answer cleanly: *can you show me the exact text this user agreed to, and prove you have not changed it since?*

The usual approach of storing something like `termsAccepted: "2.3"` on a user profile looks simple, but it creates a real audit problem. The version number tells you which release the user supposedly accepted, but it does not prove what content they saw. If someone changes the HTML and forgets to update the version, your consent record no longer matches the document shown to the user. After a few policy changes, you are left with a list of version strings and no reliable way to tell which consent is still active.

This post walks through a data-driven consent architecture built on PingOne Advanced Identity Cloud (AIC) that closes both gaps. The user's consent record links to the exact content object they saw, by database ID. Version control is enforced by the data model and a lifecycle hook, not developer discipline. When an auditor asks what `alice@example.com` agreed to on 6 August 2026, you fetch one object and get the exact HTML back.

---

## The Naming: WYSIWYA

**What You See Is What You Accept.** The name captures the core guarantee: the text rendered on screen and the text stored in the database are the same object, not copies that can diverge. The `content` field in `policyVersion` *is* what the user sees — the AM script reads it from IDM and pushes it directly to the browser callback. There is no intermediate template file to drift.


## See it in action
<video controls controlslist="nodownload" preload="metadata" width="100%">
  <source src="PingOne Advanced Identity Cloud_Consent.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
---

## The IDM Data Model

Two managed objects extend the default `managed.json` schema. Both are scoped to the `alpha` realm. Everything described here reflects the actual schema definitions from the production `managed.json`.

### `managed/policyVersion` — the Policy Catalogue

This object stores versioned legal text. It is the source of truth for what users see.

```json
{
  "name": "policyVersion",
  "schema": {
    "title": "Consent Policy Version",
    "description": "Stores versioned content for Terms of Service, Privacy Policy, etc.",
    "required": ["policyType", "content", "version", "isActive"],
    "properties": {
      "policyType":  { "type": "string",  "searchable": true,  "title": "Policy Type" },
      "version":     { "type": "string",  "searchable": true,  "title": "Consent Policy Version" },
      "content":     { "type": "string",  "searchable": true,  "title": "Consent content" },
      "isActive":    { "type": "boolean", "searchable": false, "title": "Policy Active?" },
      "publishDate": { "type": "string",  "searchable": true,  "format": "datetime" }
    }
  }
}
```

Key schema decisions worth noting:

- `content` is `searchable: true` — the field is indexed, making full-text audit queries against stored policy text possible.
- `policyType`, `version`, and `publishDate` are all searchable, supporting filtered reporting across policy history.
- `isActive` is deliberately `searchable: false`. It is a boolean flag used in point queries (`isActive eq true`), not a general-purpose filter field.
- All fields are `userEditable: false` — the UI cannot modify policy content directly.

A live object for the v1.8 terms published 6 August 2026:

```json
{
  "_id": "1ecd0c77-6387-47c4-8c0b-dfba1eceb51b",
  "policyType": "terms",
  "version": "1.8",
  "content": "<div class=\"terms-container\"><h1>General terms changed.</h1><p><strong>Version:</strong> 1.8<br><strong>Effective Date:</strong> August, 6th 2026</p>...",
  "isActive": true,
  "publishDate": "2026-08-06T00:00:00Z"
}
```

That `_id` is what gets written into every consent record. Years later, fetching that object returns the exact HTML that was in `content` on the day the user clicked Accept.

#### The Single-Active Enforcement Hook

The `policyVersion` object has both `onCreate` and `onUpdate` lifecycle hooks. When a policy object is saved with `isActive: true`, the hook automatically deactivates all other active policies of the same `policyType`:

```javascript
(function () {
    if (object.isActive === true) {
        var queryFilter = 'policyType eq "' + object.policyType + 
                          '" and isActive eq true and !(_id eq "' + object._id + '")';
        
        var oldActivePolicies = openidm.query("managed/policyVersion", { 
            "_queryFilter": queryFilter 
        });

        oldActivePolicies.result.forEach(function(oldPolicy) {
            openidm.patch("managed/policyVersion/" + oldPolicy._id, null, [
                { "operation": "replace", "field": "/isActive", "value": false }
            ]);
        });
    }
}());
```

The hook fires on both `onCreate` (publishing a brand new version) and `onUpdate` (flipping `isActive` on an existing record). You cannot accidentally have two active Terms of Service versions simultaneously — the data model enforces it at write time.

---

### `managed/consentRecord` — the Audit Trail

One record per consent event. Records are never updated in place. Each one is stamped `withdrawn` and replaced by a new `granted` record.

```json
{
  "name": "consentRecord",
  "schema": {
    "title": "Consent Records",
    "required": ["consentType", "version", "status", "timeStamp"],
    "properties": {
      "consentType":        { "type": "string",  "searchable": true,  "default": null },
      "version":            { "type": "string",  "searchable": true },
      "status":             { "type": "string",  "searchable": true,  "default": "granted" },
      "timeStamp":          { "type": "string",  "searchable": false, "format": "datetime" },
      "ipAddress":          { "type": "string",  "searchable": false },
      "withdrawnTimestamp": { "type": "string",  "searchable": false }
    }
  }
}
```

The required fields enforce that no incomplete record can be created. `status` defaults to `"granted"`. A newly created record is immediately active without a separate update step.

Two relationship fields complete the object:

**`user` relationship** (links to `managed/user`):

```json
"user": {
  "type": "relationship",
  "searchable": true,
  "returnByDefault": true,
  "reversePropertyName": "consentRecord",
  "reverseRelationship": true
}
```

`reverseRelationship: true` with `reversePropertyName: "consentRecord"` means the relationship is bidirectional. On `user`, the field `consentRecord` exposes all consent records linked to that user as a navigable array visible in the admin UI.

**`policyVersion` relationship** (links to `managed/policyVersion`):

```json
"policyVersion": {
  "type": "relationship",
  "description": "Link to the exact text version accepted.",
  "searchable": false,
  "returnByDefault": false,
  "reverseRelationship": false,
  "resourceCollection": [{
    "path": "managed/policyVersion",
    "query": { "fields": ["version"] }
  }]
}
```

`reverseRelationship: false` means it is a one-way reference — the consent record points to the policy object, but the policy object has no back-reference. Policy objects are immutable content stores, not indexes of who accepted them. `returnByDefault: false` keeps standard queries lean.

When stored:

```json
"policyVersion": {
  "_ref": "managed/policyVersion/1ecd0c77-6387-47c4-8c0b-dfba1eceb51b",
  "_refResourceCollection": "managed/policyVersion",
  "_refResourceId": "1ecd0c77-6387-47c4-8c0b-dfba1eceb51b"
}
```

### The User Connection: `consentRecord` on `user`

The `user` schema carries a reverse relationship array:

```json
"consentRecord": {
  "type": "array",
  "title": "Consent Records",
  "searchable": false,
  "returnByDefault": false,
  "userEditable": false,
  "viewable": true,
  "items": {
    "type": "relationship",
    "reversePropertyName": "user",
    "reverseRelationship": true,
    "resourceCollection": [{
      "path": "managed/consentRecord",
      "query": { "fields": ["consentType", "status", "version"] }
    }]
  }
}
```

`viewable: true` with `returnByDefault: false` means the field appears in the admin UI under the user profile but is not fetched on every user read. The query preview shows `consentType`, `status`, and `version`: enough context without loading the full HTML content.

---

## The IDM Endpoint: `consentService`

```
PATH: openidm/endpoint/consentService
TYPE: IDM Custom Endpoint Script
```

All consent logic lives here. The AM journey stays thin. The endpoint is the single controller for reading, recording, and withdrawing consent.

### Setup and Data Normalisation

```javascript
var data = request.content || {};
var journeyParams = request.additionalParameters || {};

var rawId = data.uid || journeyParams.uid || "";
var userId = rawId.trim();
var searchId = userId.indexOf('/') !== -1 ? userId.split('/').pop() : userId;

var type     = data.type     || journeyParams.type     || "terms";
var version  = data.version  || journeyParams.version  || "2.3";
var policyId = data.policyId || journeyParams.policyId;
```

AM nodes sometimes pass values in `request.content`, sometimes in `request.additionalParameters`. The endpoint checks both sources for every field and works from a single consolidated variable. User IDs can arrive as raw UUIDs or full managed object references. `.split('/').pop()` extracts the UUID either way.

### The Universal User Filter

```javascript
function getUserFilter(uid, ref) {
    return '(' + 
        'user/_ref eq "' + ref + '"' + 
        ' or ' + 
        'user/_refResourceId eq "' + uid + '"' + 
    ')';
}
```

The IDM relationship model stores two paths: `user/_ref` (full reference string) and `user/_refResourceId` (raw UUID). Depending on which index is active, either path may fail. The universal filter queries both and accepts either match.

### ACTION 1: READ — Check Consent Status

```javascript
if (request.method === "read") {
    var baseFilter = getUserFilter(searchId, fullUserRef);
    var allRecords = openidm.query(managedPath, { "_queryFilter": baseFilter });
    
    var userRecords = allRecords.result.sort(function(a, b) { 
        return new Date(b.timeStamp || 0) - new Date(a.timeStamp || 0); 
    });

    var activeConsent = userRecords.filter(function(r) {
        if (r.status !== "granted") return false;
        if (r.consentType !== type) return false;

        // Gold standard: record links to the active policy object
        var linkMatch = false;
        if (policyId && r.policyVersion && r.policyVersion._refResourceId === policyId) {
            linkMatch = true;
        }
        // Fallback: version string match (pre-migration records)
        var versionMatch = (r.version === version);
        return linkMatch || versionMatch;
    });

    return { 
        "hasConsent": activeConsent.length > 0, 
        "record": activeConsent.length > 0 ? activeConsent[0] : null 
    };
}
```

The hybrid check (`linkMatch || versionMatch`) handles migration. Users who consented before this architecture was deployed have records with a version string but no policy object link. The endpoint accepts either. Once they re-consent, their new record carries the full link.

### ACTION 2: CREATE — Record New Consent, Withdraw Previous

```javascript
if (request.method === "create" || request.action === "recordConsent") {

    var recordsToWithdraw = allUserRecords.result.filter(function(r) {
        return r.consentType === type && r.status === "granted";
    });

    recordsToWithdraw.forEach(function(rec) {
        openidm.patch(managedPath + "/" + rec._id, null, [
            { "operation": "replace", "field": "/status",            "value": "withdrawn" },
            { "operation": "add",     "field": "/withdrawnTimestamp", "value": getIsoDate() }
        ]);
    });

    var createPayload = {
        "user":        { "_ref": fullUserRef }, 
        "consentType": type,
        "version":     version,
        "status":      "granted",
        "ipAddress":   clientIP,
        "timeStamp":   getIsoDate()
    };

    if (policyId) {
        createPayload.policyVersion = { 
            "_ref": "managed/policyVersion/" + policyId 
        };
    } else {
        logger.warn("CONSENT_SERVICE: No 'policyId' provided. Record will be unlinked.");
    }

    return openidm.create(managedPath, null, createPayload);
}
```

Withdrawal happens before creation. A user always has exactly one `granted` consent record per `consentType`. Full history is preserved. No records are deleted.

### ACTION 3: PATCH — Manual Withdrawal

```javascript
if (request.method === "patch" && recordId) {
    return openidm.patch(managedPath + "/" + recordId, null, [
        { "operation": "replace", "field": "/status", "value": "withdrawn" }
    ]);
}
```

Direct withdrawal by record ID. Wire this to a self-service privacy dashboard for GDPR right-to-withdraw.

---

## The AM Journey: `consentLoginBlog`

The journey wires consent into login. Below is the actual canvas from the AIC admin UI (exported 6 August 2026).
{{< figure src="SCR_sample_ConsentJourney.png" alt="The Consent Journey" >}}


### Journey Topology

```
Start
  └─ PageNode v3.0 (Sign In — username + password)
       └─ IdentityStoreDecisionNode
            ├─ FALSE ──→ RetryLimitDecisionNode (retryLimit: 5)
            │              ├─ Retry ──→ [back to PageNode]
            │              └─ Reject ──→ AccountLockoutNode ──→ Failure
            ├─ LOCKED / CANCELLED / EXPIRED ──→ Failure
            └─ TRUE
                 └─ IdentifyExistingUserNode (identityAttribute: mail, identifier: userName)
                      └─ Check Existing Consent [ScriptedDecisionNode]
                           ├─ alreadyAccepted ──→ Update Last Login ──→ Success
                           └─ needsConsent
                                └─ Show Dynamic consent content [ScriptedDecisionNode]
                                     ├─ true (Accept)
                                     │    └─ Process and Record Consent [ScriptedDecisionNode]
                                     │         ├─ accept ──→ Update Last Login ──→ Success
                                     │         └─ error ──→ Failure
                                     └─ false (Decline)
                                          └─ PageNode v3.0 (DENIED — "ACCESS NOT ALLOWED!")
                                               └─ RetryLimitDecisionNode (retryLimit: 3)
                                                    ├─ Retry ──→ [back to Show Dyna consent content]
                                                    └─ Reject ──→ Access DENIED [User Message] ──→ Failure
```

**Notable configuration details:**

- Login retry limit: `retryLimit: 5`. Five attempts before `AccountLockoutNode` fires.
- Consent decline retry limit: `retryLimit: 3`. Three declines before hard denial. This is configurable per deployment.
- `IdentifyExistingUserNode`: `identityAttribute: "mail"`, `identifier: "userName"`. User types username, node resolves via mail attribute, writes `_id` UUID to shared state.
- `Update Last Login`: patches `custom_lastLogin` on `managed/user` via `openidm.patch()`. Convergence point for both `alreadyAccepted` and `accept` paths.

---

## The Consent Screen: What the User Actually Sees

{{< figure src="SCR_sample_EndUserConsent.png" alt="End-user consent screen showing Terms v1.8" >}}

The page header, version, effective date, all five sections, and the disclaimer footer come directly from the `content` field in `policyVersion`, rendered verbatim by `callbacksBuilder.textOutputCallback`. The Accept and Decline buttons are rendered by `callbacksBuilder.confirmationCallback`. The Ping Identity branding comes from the AIC login theme.

---

## The Three AM Scripts

### Script 1: Check Existing Consent

```javascript
(function () {
    var idmClient = (typeof idm !== 'undefined') ? idm : openidm;
    var userId = nodeState.get("_id");
    
    var policyResult = idmClient.query("managed/policyVersion", {
        "_queryFilter": 'policyType eq "terms" and isActive eq true'
    });
    
    var activeVersion = "1.0";
    var activeId      = null;
    var activeContent = "Terms and Conditions (System Default)";

    if (policyResult.result && policyResult.result.length > 0) {
        var p = policyResult.result[0];
        activeVersion = p.version;
        activeId      = p._id;
        activeContent = p.content;
    }

    nodeState.putShared("targetPolicyVersion", activeVersion);
    if (activeId) nodeState.putShared("targetPolicyId", activeId); 
    nodeState.putShared("targetPolicyText", activeContent);

    var consentResult = idmClient.query("managed/consentRecord", {
        "_queryFilter": 'user/_refResourceId eq "' + userId + 
                        '" and status eq "granted" and consentType eq "terms"'
    });
    
    var isAccepted = false;
    if (consentResult.result && consentResult.result.length > 0) {
        for (var i = 0; i < consentResult.result.length; i++) {
            var rec = consentResult.result[i];
            if (rec.version === activeVersion) { isAccepted = true; break; }
            if (activeId && rec.policyVersion && 
                rec.policyVersion._refResourceId === activeId) { 
                isAccepted = true; break; 
            }
        }
    }

    outcome = isAccepted ? "alreadyAccepted" : "needsConsent";
}());
```

### Script 2: Show Dynamic Consent

```javascript
var consentMessage = nodeState.get("targetPolicyText"); 
var options = ["Accept", "Decline"];

if (!consentMessage) {
    consentMessage = "<h3>Terms and Conditions</h3><p>Text currently unavailable.</p>";
}

if (callbacks.isEmpty()) {
    callbacksBuilder.textOutputCallback(0, consentMessage);
    callbacksBuilder.confirmationCallback(0, options, 0);
} else {
    var selectionIndex = callbacks.getConfirmationCallbacks().get(0);
    action.goTo(selectionIndex === 0 ? "true" : "false");
}
```

Uses AIC's Next-Generation scripting API. No `action.send()` call. Callbacks in the builder transmit automatically on first-pass completion.

### Script 3: Process and Record Consent

```javascript
(function () {
    var idmClient = (typeof idm !== 'undefined') ? idm : openidm;
    try {
        var userId   = nodeState.get("_id");
        
        // Force-cast: shared state values arrive as Java objects in the Next-Gen engine
        var rawId    = nodeState.get("targetPolicyId");
        var policyId = rawId ? String(rawId) : null;
        
        var version  = nodeState.get("targetPolicyVersion") || "1.8";
        var clientIP = nodeState.get("consentIP") || "0.0.0.0"; 

        var payload = {
            "uid":      userId,
            "type":     "terms",
            "version":  version,
            "policyId": policyId,
            "ip":       clientIP
        };

        idmClient.action("endpoint/consentService", "recordConsent", {}, payload);
        outcome = "accept";
    } catch (e) {
        logger.error("Process Consent Error: " + e.toString());
        outcome = "error";
    }
}());
```

The `String(rawId)` cast is necessary. Shared state values are Java objects in the Next-Gen environment. Passing one into `JSON.stringify()` produces `{}` — the UUID disappears silently. The cast guarantees a JS primitive string before serialisation.

**On client IP capture:** The script reads `nodeState.get("consentIP")` and falls back to `"0.0.0.0"`. The `ipAddress` field exists on the `consentRecord` schema. Populating it requires a node upstream of the consent screen to capture the client IP and write it to shared state. This is a planned addition requiring no schema changes.

---

## What an Audit Actually Looks Like

```
GET /openidm/managed/consentRecord
  ?_queryFilter=user/_refResourceId eq "<alice-uuid>" and consentType eq "terms"
  &_sortKeys=-timeStamp
```

The response lists every consent event for Alice, newest first. The `granted` record for v1.8 carries `policyVersion._ref` pointing to `managed/policyVersion/1ecd0c77-...`. Fetch that object and you get the exact HTML back. The `withdrawnTimestamp` on older records shows when each version was superseded.

No reconstruction. No reliance on file system snapshots. No risk of template drift between signing date and audit date.

---

## Operational Patterns

**Publishing a new policy version:** Create a new `policyVersion` record with `isActive: true`. The `onCreate` hook deactivates all other active records of the same `policyType` automatically. Every user who logs in after that point sees the new text.

**Multiple consent types:** `policyType` / `consentType` isolates each stream. Marketing consent is entirely independent from Terms. Each type has its own active policy object and its own consent record per user.

**Self-service withdrawal:** The `PATCH` action accepts a `recordId` and sets `status` to `withdrawn`. The `consentRecord` relationship on `user` makes full consent history visible from the user profile in the admin UI.

**Legacy migration:** No data migration required. The hybrid check accepts records with no `policyVersion` link as long as the version string matches. Records gain the full link on next re-consent.

**Debugging:** The `?action=list` path returns a user's full consent history without filtering. Combined with `logger.error` statements in the Process Consent script, you can trace any consent event from AM journey through to the IDM record.

---

## Summary

| Capability | How |
|---|---|
| **Proof of content** | `policyVersion._ref` links every consent record to the exact HTML object the user saw |
| **Single active policy** | `onCreate`/`onUpdate` hook on `policyVersion` deactivates previous versions automatically |
| **Single active consent** | Auto-withdrawal in `consentService` stamps previous `granted` records before creating the new one |
| **Zero-deployment policy updates** | Legal creates a new `policyVersion` object; re-consent triggers on next login |
| **Re-consent on decline** | Configurable `RetryLimitDecisionNode` on decline path. Default 3 retries before hard denial. |
| **Backward compatibility** | Hybrid check accepts version-string-only records alongside full policy-linked records |

The AM journey handles authentication and user experience. The IDM endpoint handles all consent state logic. The IDM data model handles durability, auditability, and lifecycle. Each layer has one job, and none of them depends on a file staying unchanged over time.
