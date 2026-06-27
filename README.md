# Grand Central Payments Connector - Case Study

**Parth · Forward Deployed Product Manager · Backbase**

---

## The case

A banking iPaaS connector, four weeks from go-live, is returning payment failure responses the bank's ops team cannot act on. Some are structured and routable. Others are blank, generic, or raw core codes with no translation. The ops team cannot tell, from the failure itself, where in the flow it originated.

Engineering's position: the connector is passing through whatever the core or rail returns, and behaving correctly per the v3 spec. The bank's position: the platform is not ready to go live. Neither side had hard evidence yet.

The case question asks four things:

1. **Diagnose** - how do you get to the truth of where the problem lies, and what does that investigation look like?
2. **Product thinking** - this is the third time the same pattern has surfaced. What does that signal, and what do you do with it beyond fixing it for this bank?
3. **Trade-off** - engineering proposes a quick client-side workaround to make the go-live date. Do you take it? What are the implications?
4. **Stakeholder communication** - what do you tell the CTO today, before you have a full answer?

---

## What this repo contains

### The presentation

`index.html` - a 15-slide interactive deck covering the full case. Navigate with arrow keys, Space, or the progress rail along the bottom. Press 1-6 to jump to an act.

The six acts: Setting the Scene, The Crisis, The Investigation, The Decision, The Fix, The Resolution.

### The investigation artifacts

`resources/meridian_uat_cycle3_transactions.csv` - 500 synthetic UAT transactions from cycle 3, with full request and response payloads. 47 failures across SCT, SCT Inst, and SWIFT. Includes origin columns for cross-referencing with the analysis in the deck.

`resources/meridian_uat_cycle3_connector_logs.csv` - 301 connector log lines across the 47 failures. Contains the two smoking-gun entries: the unhandled timeout on SCT Inst, and the no-mapping passthrough on the T24 core code.

### The API contracts

Three versions of the Grand Central Payments Connector OpenAPI spec, one per bank implementation. The asymmetry between the request side and the failure response side across versions is the core of the technical finding.

| File | Version | Bank | Rails | Failure response |
|---|---|---|---|---|
| `gc-payments-connector-v1.yaml` | v1 | Apex Banking, IE | SCT | status + optional code |
| `gc-payments-connector-v2.yaml` | v2 | Sparkasse Mittelrhein, DE | SCT + SCT Inst | unchanged from v1 |
| `gc-payments-connector-v3.yaml` | v3 | Meridian Bank NV, NL | SCT + SCT Inst + SWIFT | unchanged from v1 |

Rendered HTML views of each (no raw YAML):

- `resources/api-v1.html`
- `resources/api-v2.html`
- `resources/api-v3.html`

The v3 rendered view annotates all four failure response examples from the UAT data and explains which are actionable and why.

### The proposed fix

`resources/gc_error_mapping_config_v3.json` - the error mapping configuration that is the go-live fix. Maps raw codes to a canonical envelope: origin, code, reason.

Three sections:

- `rail` - SEPA scheme reason codes and SWIFT rejection codes. Standard across all banks. 98 entries.
- `connector` - connector-generated codes including timeout, validation errors, and connection failures. Standard. 12 entries.
- `core` - bank core codes. Meridian's T24 mappings. 33 entries. Bank-owned.
- `fallback` - applied when a raw code has no match. Always returns a structured envelope. Never blank.

The config does not carry retry flags, SLA targets, or ops actions. Those are the bank's operational policy, not the connector's contract.

---

## The key finding

The request side mirrors ISO 20022 pain.001 and grew carefully with each new rail: 7 fields in v1, 12 in v2, 20+ in v3. The failure response stayed at status + optional code across all three versions.

Each bank that hit the gap patched it in their own build. The patch never reached the shared baseline:

- Apex Banking (v1): SEPA reject codes not translated. Fixed in the Apex build only.
- Sparkasse Mittelrhein (v2): connector timeout returns empty. Fixed in the Sparkasse config only.
- Meridian (v3): both of the above, rediscovered, plus SWIFT failure modes no one had standardised.

The go-live fix is a config expansion - no code change, no regression. The platform fix is v3.5: making `origin` a required field so the contract treats failure as a first-class concern.

---

## What was rejected and why

**Bank-side interpreter:** Meridian builds a client-side response cleaner on the T24 side. Low risk to the date, but makes Meridian own Backbase's gap permanently and leaves the next bank to rediscover it. The third instance of a pattern is the moment to break it, not repeat it.

**Connector code change now:** Rebuilds response logic in code, 20 days from go-live. High regression risk, full sprint disruption. The fix is addressable through config alone.

---

*Synthetic case study. Meridian Bank NV, Apex Banking, and Sparkasse Mittelrhein are fictional. All data is generated for this case.*
