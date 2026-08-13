# Hearsay

**Provenance for the AI Passport write path.**

The AI Passport stores a model's guess about you the same way it stores what you typed. Hearsay adds a provenance class to every field, blocks apps from upgrading a guess into a fact, quarantines inferences that land in sensitive categories, and makes a correction travel to every app that already read the old value.

Submitted to the [AI Passport Ideathon](https://ai-passport-ideathon.devpost.com) by Egoist Machines. **Identity track, Build lane.**

- Devpost project: https://devpost.com/software/hearsay-i6w9sh
- Walkthrough video: https://www.youtube.com/watch?v=UrbMqHbBhRY
- One-page deck: [`artifact/hearsay-deck.pdf`](artifact/hearsay-deck.pdf)

---

## The finding

The passport's read path has five controls. The write path has one.

| Control | Read path | Write path |
|---|---|---|
| What is named | Exact fields | A free-text string |
| Why | A declared purpose | Nothing |
| How long | A duration | Forever |
| User decision | Approve, edit, deny | One tap |
| Evidence | Not applicable | Nothing |
| Category | Implied by field | Nothing |

Inside one app, a bad inference dies with the app. Inside a passport, it becomes globally authoritative. The passport does not only carry context. It launders inference into fact.

## Four classes

A class is a claim about **evidence**, not about truth. It verifies nothing. It preserves the ability to evaluate a fact later, and puts the cost of a bad guess back on whoever made it.

| Class | Meaning |
|---|---|
| `declared` | The holder typed it |
| `attested` | A third party signed it |
| `observed` | A hard event exists in a connector |
| `inferred` | A model concluded it, with source, evidence, confidence and expiry |

## Four rules

1. **No upward laundering.** A write inherits the lowest class among its inputs. An app that reads an `inferred` field cannot re-propose the result as `observed`. This is taint tracking for personal facts, and declassification requires the holder, not the app.
2. **Inferences decay.** Every inferred field carries a mandatory TTL. Renewal requires evidence from a source the original write did not use, so an app cannot refresh its own guess forever.
3. **Sensitive categories are quarantined.** Apps cannot request inferred values in a sensitive category at all. The API refuses rather than asking the user to approve. Quarantined items are never bundled, never readable by any app under any grant, and deleted after 30 days if untouched.
4. **Corrections travel.** Editing or deleting a field broadcasts a retraction to every app in that field's receipt log, with per-app state visible to the holder.

## The change, in full

Today:

```
suggest: "managing a chronic sleep condition"
```

Under Hearsay:

```json
POST /passport/suggest
400  PROVENANCE_REQUIRED

{
  "field":        "health.conditions",
  "value":        "chronic sleep condition",
  "provenance":   "inferred",
  "source_app":   "app_sleepcoach_v2",
  "derived_from": ["whoop.hrv_30d", "calendar.cancellations_30d"],
  "confidence":   0.61,
  "ttl":          "P30D"
}

409  QUARANTINED
{
  "category":          "sensitive.health",
  "shared":            false,
  "readable_by_apps":  false,
  "bundling":          "prohibited",
  "auto_delete_at":    "+30d"
}
```

What a requesting app receives on a read:

```json
{
  "value":       "prefers morning-free scheduling",
  "provenance":  "inferred",
  "asserted_at": "2026-07-02",
  "expires":     "2026-08-01"
}
```

The class and the age travel. The evidence never does. Only the holder sees why.

## Schemas

JSON Schema, draft 2020-12. The conditional blocks are where the rules live.

| File | What it covers |
|---|---|
| [`schema/suggest.request.schema.json`](schema/suggest.request.schema.json) | The write. `provenance` is required; each class conditionally requires its own evidence fields. |
| [`schema/suggest.responses.schema.json`](schema/suggest.responses.schema.json) | `201 ACCEPTED`, `400 PROVENANCE_REQUIRED`, `403 CLASS_NOT_PERMITTED`, `409 QUARANTINED`, plus the sensitive-category list. |
| [`schema/field.read.schema.json`](schema/field.read.schema.json) | What a requesting app receives, and the retraction shape with per-app state. |

Validate with any 2020-12 validator, for example:

```bash
npx ajv-cli validate -s schema/suggest.request.schema.json -d example.json --spec=draft2020
```

## Threat model

| Attack | Mitigation |
|---|---|
| Laundering through derivation | Label inheritance. A derived write cannot exceed the lowest class among its inputs. |
| Expiry evasion by self-refresh | Renewal requires evidence from a source the original write did not use. |
| Consent fatigue | Rate-limited `suggest`, weekly batched review, quarantine defaults to deny and never nags. |
| Third-party injection via the public profile page | Inbound third-party content caps at `inferred` and cannot enter a sensitive category. |
| Prompt injection through connector content | Same. Connector text is evidence for an inference at best, and cannot elevate itself. |
| Class misreporting by an app | Sampled audits of `derived_from`, directory downgrade, loss of write access. Inheritance makes the lie useful once. |

Two risks belong to this design. The sensitivity classifier is itself an inference, so it fails toward quarantine: a false positive costs one extra tap, a false negative costs a job or an insurance policy. And the quarantine tray would become the most sensitive object in the system if it retained what it rejected, which is why untouched items are deleted rather than archived.

One risk this does not solve: a single record read by everything is the highest-value target in the system, and provenance does nothing about concentration.

## Version one

Roughly two weeks inside the existing architecture.

1. `provenance` becomes a required enum on the suggest endpoint. Writes that omit it are rejected.
2. A sensitivity classifier and a quarantine tray. Deny by default, no bundling, no app read path, auto-delete at 30 days.
3. A provenance badge on every field in the passport UI and inside every receipt.

**Deliberately not in v1:** cryptographic attestation or an issuer network (`attested` is a reserved slot, not a deliverable); the full retraction acknowledgment protocol beyond broadcast and per-app state; automated auditing of `derived_from`; anything zero-knowledge.

## On revocation, honestly

A retraction cannot un-read data and cannot un-train a model. What it does is stop future reads, timestamp the withdrawal, and make any use after that timestamp a documented breach rather than a dispute about memory. It converts an unenforceable technical promise into an enforceable contractual one, and the lever behind it is market access.

## Artifact

| File | |
|---|---|
| [`artifact/hearsay-deck.pdf`](artifact/hearsay-deck.pdf) | Six pages, the full walkthrough |
| `artifact/01-cover.png` | The problem in one screen |
| `artifact/02-record-before-after.png` | The record today, and under Hearsay |
| `artifact/03-panel-a-today.png` | Read path against write path |
| `artifact/04-panel-b-hearsay.png` | The same call, rejected then quarantined |
| `artifact/05-panel-c-receipt.png` | Receipt and retraction |
| `artifact/06-four-classes.png` | Four classes, four rules, one enum |

---

This entry is for Maya, who needs to prove to every app reading her passport that a claim about her health was a model's guess and not a fact she stated, so she can correct it once and have the correction travel, without ever revealing the condition itself.
