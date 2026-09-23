# Microsoft API Hosted Runner

This plugin consolidates stable, read-only Microsoft pull APIs into one
Gravwell Hosted Runner process. Production deployment uses one named
`Microsoft` stanza per exact `Api` selector. Stanzas may reference the same
tenant, application, and `Client-Secret-File`, but each has an independent
UUID, poll lifecycle, state key, permission result, and completion log. A
blocked product API therefore cannot turn the multi-selector deployment into one
all-or-none polling unit.

The production command is `hosted/runner`; Microsoft is registered
as one native Hosted Runner plugin in that command. It is not a Gravwell
Fetcher. Its default configuration and
overlay locations are:

- `/opt/gravwell/etc/hosted_runner.conf`
- `/opt/gravwell/etc/hosted_runner.conf.d`

Every secret is loaded from a file. `Client-Secret-File` is reread when a token
is renewed, while the Gravwell ingest secret remains owned by the exact native
`[Global]` configuration. Neither secret is accepted as a Microsoft stanza
value, command argument, or environment variable.

## Configuration parameters

| Parameter | Type | Required | Default | Notes |
|---|---|---:|---|---|
| `Ingester-UUID` | UUID | yes | none | Must be unique and non-zero at runtime |
| `Tenant-ID` | string | yes | none | Entra tenant |
| `Client-ID` | string | yes | none | Entra application |
| `Client-Secret-File` | path | yes | none | Read-only file; secret is never logged |
| `Api` | string | yes | none | Exactly one exact catalog selector per production stanza; the parser accepts repeats only for compatibility |
| `Subscription-ID` | UUID, repeatable | conditional | none | Azure subscription/Resource Graph APIs |
| `Tag-Name` | tag | no | exact dataset tag | One resolved API only; incompatible with `Tag-Prefix` |
| `Tag-Prefix` | string | no | none | Multi-API override; incompatible with `Tag-Name` |
| `Tag-Schema` | enum | no | `legacy` | `legacy` preserves existing exact tags; `consolidated` opts into 15 semantic destinations |
| `Lookback` | hours or duration | no | `24h` | `24h`, `7d`, `1d12h`, or legacy integer hours; 1–672 hours |
| `Requests-Per-Minute` | integer | no | 10 | 1–6000 |
| `Request-Interval` | seconds | no | 3600 | 60–9223372036; must fit the native duration |
| `Page-Size` | integer | no | 10 | 1–1000 |
| `Max-Pages` | integer | no | 1 | 1–1000 |
| `Overlap` | seconds | no | 900 | 0–86400 |
| `Max-Retries` | integer | no | 3 | 1–10; omitted or zero uses the compatibility default |
| `Ingest-Unchanged` | boolean | no | false | Explicit re-emission, including repeated occurrences; Fabric retention rescans always suppress unchanged versions |
| `Normalization` | enum | no | `disabled` | `disabled`/`false` preserves the compact selected Microsoft record; `enabled`/`true` emits `microsoft-normalization-v1` to a separate `-normalized` tag |
| `Graph-Host` | HTTPS URL | no | `https://graph.microsoft.com` | Endpoint override; OAuth audience remains catalog-defined |
| `ARM-Host` | HTTPS URL | no | `https://management.azure.com` | Endpoint override; OAuth audience remains catalog-defined |
| `Defender-Host` | HTTPS URL | no | `https://api.security.microsoft.com` | Endpoint override; OAuth audience remains catalog-defined |
| `PowerBI-Host` | HTTPS URL | no | `https://api.powerbi.com` | Endpoint override; OAuth audience remains catalog-defined |
| `Auth-Host` | HTTPS URL | no | `https://login.microsoftonline.com` | OAuth endpoint override |
| `Preprocessor` | string, repeatable | no | none | Named standard Gravwell preprocessors |

The Hosted Runner's `[Global]` is the unmodified Gravwell
`config.IngestConfig`; vendor fields never appear in it. The executable
accepts every standard field implemented by the pinned Gravwell SDK.

The current catalog targets Microsoft's commercial cloud. Changing a host does
not change its OAuth audience or establish sovereign-cloud compatibility.


## Supported selectors

| Selector | Included datasets |
|---|---|
| `azure` | Activity Log, Resource Graph inventory, and Policy States |
| `entra` | Directory audits, sign-ins, risk detections, risky users, and users/groups/applications/service principals/devices/Conditional Access inventory |
| `defender` | All listed Defender for Cloud, XDR, and Endpoint datasets |
| `defender-cloud` | Defender for Cloud alerts, assessments, assessment metadata, subassessments, secure scores, secure-score controls, and regulatory-compliance standards |
| `defender-xdr` | Defender XDR incidents, alerts v2, and Graph secure scores |
| `defender-endpoint` | Devices, vulnerabilities, software, recommendations, and organization exposure score |
| `intune` | Audit events and managed devices |
| `fabric` | Tenant Activity Events |
| `all` | Every dataset above |

Every individual dataset name is also a valid `Api` value. The catalog in
`catalog.go` is authoritative for names, endpoints, OAuth audiences, required
permissions, stable IDs, source timestamps, and default exact Gravwell tags.
Group aliases such as `all` remain useful for discovery and targeted tests,
but the generated production examples deliberately expand them into 94
independent stanzas.

### Optional consolidated tag schema

Omitting `Tag-Schema` or setting it to `legacy` preserves every current tag.
`Tag-Schema="consolidated"` is an explicit migration mode that routes the
catalog into 15 stable families: Azure activity, inventory, and policy;
Defender for Cloud and Endpoint; Defender XDR alerts, incidents, hunting, and
posture; Entra directory, governance, and security; Microsoft 365; Intune; and
Fabric. Existing `_source`, `_recordType`, `_product`, endpoint, and API-version
intrinsics preserve the selector and source contract inside each destination.
Checkpoint keys remain selector-based, so changing tag schema does not rename
state. Do not enable consolidated mode with current kits until their tag
references are migrated and validated; rollback is the same configuration with
`Tag-Schema="legacy"` and the original UUID/state.

### Defender for Cloud tag ownership migration

The seven `defender-cloud-*` selectors now default to product-owned
`microsoft-defender-cloud-*` tags. Selector names and state keys did not
change, so the tag migration itself does not replay records or create a second
delivery path. Do not configure the same selector in parallel old-tag and
new-tag stanzas. A one-selector stanza can temporarily set an explicit legacy
`Tag-Name` for a controlled rollback, but the production default and kit/well
contract is the product-owned tag. Normalized mode appends `-normalized` to the
new default.

| Selector | Previous default | Product-owned default |
|---|---|---|
| `defender-cloud-alerts` | `azure-defender-alerts` | `microsoft-defender-cloud-alerts` |
| `defender-cloud-assessments` | `azure-defender-assessments` | `microsoft-defender-cloud-assessments` |
| `defender-cloud-assessment-metadata` | `azure-defender-assessment-metadata` | `microsoft-defender-cloud-assessment-metadata` |
| `defender-cloud-subassessments` | `azure-defender-subassessments` | `microsoft-defender-cloud-subassessments` |
| `defender-cloud-secure-scores` | `azure-defender-secure-scores` | `microsoft-defender-cloud-secure-scores` |
| `defender-cloud-secure-score-controls` | `azure-defender-secure-score-controls` | `microsoft-defender-cloud-secure-score-controls` |
| `defender-cloud-regulatory-standards` | `azure-defender-regulatory-standards` | `microsoft-defender-cloud-regulatory-standards` |

## State and framing

Version 1.3.2 reads `entra-pim-assignments` as a complete, content-deduplicated
snapshot. The [PIM list API](https://learn.microsoft.com/en-us/graph/api/rbacapplication-list-roleassignmentscheduleinstances?view=graph-rest-1.0)
does not document `$top`; page size is server-controlled and the runner follows
`@odata.nextLink` within `Max-Pages`. No moving `startDateTime` filter is sent:
current assignments may have old or null start dates. The original record and
source start date remain intact; null dates use the observation time. Existing
state keys and tags are preserved. Previously omitted assignments are emitted
once on the next successful complete poll. If a continuation exceeds the page
budget, the poll fails without advancing state; increase `Max-Pages` and retry.

Version 1.3.1 removes fixed ten-row hunting sampling. Each hunting query requests
one more row than the `Page-Size * Max-Pages` record budget, capped at 99,999
accepted records below Microsoft's 100,000-row ceiling. All returned rows within
budget are retained. Exceeding the budget, response-size limit, or a malformed
response fails the poll without advancing its checkpoint. Hunting does not offer
OData pagination: reduce the event lookback or increase the budget before retrying.
An already-backlogged checkpoint also requires an explicitly planned recovery;
changing Lookback alone does not override persisted watermarks.

`microsoft-defender-vulnerability` is a current `DeviceTvmSoftwareVulnerabilities`
snapshot keyed by device, software vendor/name/version, and CVE. It has no native
Timestamp or ReportId and is distinct from the `/api/vulnerabilities` catalog
under `microsoft-defender-vulnerabilities`. Its corrected composite identity can
emit the current findings once after upgrade; existing state files are retained.

Equivalent selector aliases in one stanza or group resolve to one collection
contract. Individually configured legacy selectors keep their state keys. Do not
enable equivalent aliases in separate stanzas; the complete example includes one
representative per identical request, authorization, state mode, and destination.

Append streams use a source-timestamp watermark with bounded overlap and
event-ID deduplication. The first successful empty poll stores its initial
lookback lower bound; later empty polls retain established progress. Nonempty
temporal polls advance monotonically to the greatest selected source timestamp,
bounded by the requested window end. They do not advance to the wall clock merely
because a poll succeeded. Late visibility after nonempty progress is covered only
within the configured overlap and the vendor's retention; this is not an unlimited
late-arrival guarantee. Existing persisted progress takes precedence over changed
lookback. A historical gap caused by an older version's watermark is not silently
rewound.

Fabric activity uses a separate durable scan anchor because Microsoft's
[activity API](https://learn.microsoft.com/en-us/rest/api/power-bi/admin/get-activity-events)
only accepts dates in the preceding 28 days, split into single UTC days. Each
poll revisits the original anchored range still within that retention horizon,
even after newer events advance the event watermark. An empty poll retains the
event watermark; it records successful scan bounds separately. Changed Lookback
does not reset the anchor. The lower transport bound is rounded up to the next
millisecond only at the retention edge, and the upper bound is truncated to a
millisecond so rounding cannot create a span over 28 days. Other boundaries keep
the existing outward rounding for submillisecond ties.

As history expires, the runner warns with the original anchor and available
lower bound; successful state records `fabric_scan.anchor`, `available_since`,
and `through`. These are scan metadata, not proof that expired or still-hidden
events were delivered. This rescans up to 28 days (up to 29 UTC-day slices plus
pagination) per poll. The service's 200-request/hour limit, configured pacing,
page/byte budgets, API clock differences and request-time aging still apply.
The client retains state on API errors; the next attempt recomputes the available
range. No polling algorithm can recover data that expires before a successful
request or arrives after retention.

Upgrade reads the existing state key and JSON. When scan metadata is absent,
the prior event watermark minus overlap, or an existing pending start, becomes
the Fabric anchor. Earlier pre-upgrade history cannot be inferred or recovered
automatically. An expired pending window is rebased to its still-available part;
if wholly expired it resumes the available anchored range through the current
poll time. Acknowledged version receipts remain usable for records still
returned. Only a fully successful poll commits scan metadata and event progress.
Partial receipts remain separate. Downgrading to a version without this policy
can reintroduce the old quiet-window failure; preserve state and upgrade again
rather than clearing checkpoints. Public tags, config options, identity and
raw/normalized state namespaces are unchanged by the Fabric window policy.

Fabric checks deduplication capacity before delivering any fetched entries.
Every identity in the current scan is protected from eviction. If the scan
contains more than 500,000 distinct identities, the poll logs a capacity error,
writes nothing, retains its prior durable state, and retries on Request-Interval.
It can recover when the available result falls within capacity; the error does
not mean the complete history was ingested. Absent identities can age out or
make room, so a previously absent identity returning later can replay. This
bounded in-memory state is not a high-volume archival guarantee.

Fabric rescans discover late-visible events across retained history. They
always suppress unchanged versions even when Ingest-Unchanged is true; new
and changed versions still emit. This explicit exception prevents replaying
the complete retained window every poll. Other selectors retain intentional
unchanged-record re-emission.

Inventory and non-temporal APIs always refresh their complete lists;
the shared legacy Lookback setting does not apply a temporal filter to them.
Lifecycle and inventory streams also retain a content digest so changed records
are emitted while semantically unchanged records are suppressed by default.
The digest canonicalizes object-key ordering. Two selector-specific policies
omit only volatile Microsoft enrichment from the change fingerprint:
`defender-cloud-assessments` omits
`properties.additionalData`, and `defender-cloud-secure-score-controls` omits
`properties.definition.properties.assessmentDefinitions`. The complete raw
record is still emitted, and assessment status plus secure-score/control state
remain fingerprinted. The production builder writes through the native ingest
muxer and requires successful `SyncContext` before recording delivery receipts.
Acknowledged partial progress is saved and flushed through the native state
callback in batches of at most 64 entries and before returning a later record
write failure. Retries retain the incomplete poll's time window (subject to the
documented Fabric retention rebase) and skip its
acknowledged record versions, including with `Ingest-Unchanged=true`. Repeated
identical occurrences requested by that flag have separate acknowledgment
counts: an acknowledged first occurrence cannot suppress a second occurrence,
and retries skip only the number already acknowledged. Predecessor boolean
receipts remain readable and acknowledge one occurrence. A temporarily shorter
retry result retains the acknowledged count while that version remains present.
Only a
complete poll replaces the dataset watermark and committed snapshot. Completed
polls honor intentional re-emission except for the Fabric rescan rule above. A write, cancellation,
synchronization, or state error leaves the completed watermark unchanged.
Pending receipts are bounded to versions present in the current fetched result;
the previous state key and completed record layout remain readable.

This is at-least-once delivery. Up to the current 64-entry batch can be uncertain
when synchronization or state persistence fails or the process stops between
delivery and receipt commit. An entry whose write itself returns an error may
also have been partially accepted. These uncertain entries can replay on another
attempt; there is no exactly-once or lifetime duplicate-count guarantee during
repeated failures of that boundary. Previously confirmed receipts do not replay
merely because a later record repeatedly fails. SDK synchronization is not a claim of searchable backend data;
backend acceptance and searchability require separate live verification.

Full-list collection emits observed changes, not deletion events. Missing objects
do not produce tombstones. Snapshot-mode histories are replaced after a complete
refresh, so a disappeared object that reappears is emitted again. Legacy lifecycle
full-list histories retain their digests subject to the documented bounded state
retention (35 days and at most 500,000 identities); absence alone is not a state
change. Access-review instances and Intune managed devices are full snapshots:
their scheduled end and last successful device-sync timestamps do not establish
complete mutation clocks. See Microsoft's [accessReviewInstance schema](https://learn.microsoft.com/en-us/graph/api/resources/accessreviewinstance?view=graph-rest-1.0)
and [managed-device update contract](https://learn.microsoft.com/en-us/graph/api/intune-devices-manageddevice-update?view=graph-rest-1.0).
Their existing state keys, source timestamp fields and tags remain unchanged;
upgrade can emit previously unobserved older current objects once.

Parent/child streams (`defender-users` and `entra-access-reviews`) identify a
child with an unambiguous JSON pair `[parent ID, child ID]`. Raw vendor records
remain unchanged; normalized `sourceId` uses that pair. This prevents the same
account observed on different machines, or child IDs under different definitions,
from overwriting each other's state. Earlier unscoped identities and receipts
cannot be assigned to a particular parent safely: upgrade emits each current
parent/child version once, then resumes normal deduplication. Existing state keys
remain readable; do not clear checkpoints. Downgrading can reintroduce identity
collisions and repeated delivery.
API page cursors live for one fetch;
all pages are fetched before any entry is written. A page-cap failure retains
progress and can recover on a new poll or a larger supported page budget. It does
not automatically drain an arbitrarily large result over multiple poll cycles.

API page envelopes and continuation fields are transport metadata. Each vendor
record is extracted, compacted with `json.Compact`, and written unchanged as
exactly one Gravwell entry. The plugin never ingests an array as one entry,
pretty-prints JSON, or adds synthetic `tag`, `Vendor`, `Product`, or collector
fields to the default record. In legacy mode the tag also identifies the
dataset; in consolidated mode the `_source` and `_recordType` intrinsics retain
that exact identity inside the semantic family tag.

`Normalization=enabled` is an explicit alternate contract. It emits one
compact `microsoft-normalization-v1` object containing `contractVersion`,
`product`, `recordType`, `selector`, `sourceId`, `sourceTimestamp`, and the
selected source object under `record`, with the duplicate-key limitation below.
It appends `-normalized` to the
resolved tag and uses a separate state namespace, so changing modes cannot
silently suppress the first normalized snapshot. The original wrapper fields,
tag suffix, and checkpoint namespace remain compatible. Additive
`fieldNormalizationVersion="microsoft-fields-v1"` identifies the field rules;
`canonical` contains reconciled aliases and `normalizationCollisions` records
disagreements. Tag consolidation remains controlled separately by `Tag-Schema`.

Rules use the first nonempty string or number, with an existing canonical field
first. They preserve vendor fields and values under `record` subject to the
map-decoding limitation: if an object repeats a key, the final occurrence wins.
Raw compact output preserves repeated keys and numeric spelling, but record-ID,
timestamp extraction and the semantic deduplication digest also use last-key-wins
decoding. Changes only to earlier duplicate-key occurrences can therefore be
suppressed by deduplication. Normalization is not byte-lossless. Unsupported
types are preserved in the original record and do not create an alias. Collision
entries name the canonical field, selected source path, and disagreeing source
paths; original conflicting values remain available under `record`.

| Canonical path | Ordered source paths | Selector coverage |
|---|---|---|
| `canonical.recordId` | `recordId`, catalog `IDField` | Each selector with a single documented ID field; composite IDs are not invented |
| `canonical.recordTimestamp` | `recordTimestamp`, catalog `TimeField` | Each selector with a documented timestamp field; values remain unmodified |
| `canonical.userName` | `userName`, `userPrincipalName` | `entra-signins` |
| `canonical.sourceIp` | `sourceIp`, `ipAddress` | `entra-signins` |
| `canonical.userName` | `userName`, `initiatedBy.user.userPrincipalName` | `entra-directory-audits`; targets are never substituted for the actor |
| `canonical.actionName` | `actionName`, `activityDisplayName` | `entra-directory-audits` |

These source meanings are documented in the Microsoft Graph
[signIn schema](https://learn.microsoft.com/en-us/graph/api/resources/signin?view=graph-rest-1.0)
and [directoryAudit schema](https://learn.microsoft.com/en-us/graph/api/resources/directoryaudit?view=graph-rest-1.0).
For normalized records, extraction must use the explicit `canonical.*` paths
(for example `json canonical.userName as userName`) or `record.*` paths for
vendor fields. Existing raw-tag extractors are not compatible with the wrapper.
Leave normalization disabled until the consuming normalized-tag extractor and
query have been verified on the target backend.

## Failure cadence

Each selector/subscription poll limits API responses to 64 MiB individually,
128 MiB in total, and 1000 total data-request attempts, including nested and
multi-day pagination. OAuth response bodies and credential files are limited
to 1 MiB. Exceeding a bound fails the poll and retains its checkpoint. These
are fixed safety bounds; `Max-Pages` and `Max-Retries` may impose lower limits.
After retry exhaustion, an origin's server cooldown remains active for later
polls and recreated jobs. Actual origin eligibility is stored separately from
source delivery state under the job-wide `microsoft/retry-origins-v1` key,
independent of normalization mode. The immediately preceding raw and normalized
retry keys are merged conservatively: the latest deadline wins, and an
unrepresentable hint remains blocked. The normalized predecessor is retained
for rollback; source delivery histories are unchanged. A migration read, decode,
or synchronization failure stops the poll before requests. No credentials or
response bodies are stored. Both data and OAuth origins use this rule.
Redirects are rejected and pagination retains its
original service origin.

The Microsoft client applies bounded request retries before returning a poll
failure. HTTP 429 and 5xx responses preserve the actual `Retry-After` eligibility,
including representable numeric values and far-future HTTP dates. Ten minutes
bounds the shared in-cycle cooldown-wait budget; it never shortens a vendor hint.
An origin beyond that budget returns a visible deferral immediately. Later polls
reload its eligibility and cannot issue an early request. Other origins continue;
an unavailable authentication origin necessarily defers selectors needing a new
token, while already-valid cached tokens can still be used.

Numeric hints beyond the durable timestamp range (through year 9999), including
numeric overflow, persist an explicit blocked-origin marker and return a visible
error. They never wrap into an immediate retry. Such a marker requires an
operator to correct the affected origin's retry-state entry and recreate the job
after resolving the invalid server response; do not clear source checkpoints. Representable future
eligibility becomes available normally as time passes. The terminal attempt
records eligibility and returns an error without sleeping. Missing or invalid
non-numeric headers use bounded exponential delay. Storage failures are returned
as operational errors and retain in-memory eligibility; persistence has the same
native durability limits as other state. Downgrading to a version that does not
load this additive retry state loses this guarantee.

A source-poll failure that remains after
those retries—including an unavailable or unlicensed 400, unauthorized 401,
or forbidden 403—is logged as `Microsoft API selector poll failed` with the
exact selector, whether it was subscription-scoped, and the safe error text.
Tenant IDs and subscription IDs are not added to that log event.

After logging a source-poll failure, `Handle` returns a successful
`Continuation` for the configured `Request-Interval`. This is intentional:
the generic Hosted Runner adapter otherwise discards that continuation for any
non-nil error and retries after its fixed 30-second `JobErrorDelay`. Persistent
permission and entitlement failures therefore poll every 3600 seconds by
default, rather than every 30 seconds. Other selectors in the same compatible
multi-selector stanza are still attempted, although production examples use
one selector per stanza.

Context cancellation and local operational failures are not converted into a
successful poll. State reads/writes, tag negotiation, preprocessing, and
ingest writes remain non-nil errors, retain the framework's operational retry
path, and never advance the failed selector's state. There is deliberately no
new global or Microsoft stanza parameter for this behavior.

The Shared and Standalone Fetchers do not use the Hosted Runner job adapter.
Their loops already log the joined poll error and unconditionally perform a
cancellation-aware sleep for `Request-Interval`; a failed poll cannot select a
30-second framework retry. Their cadence logic required no code change in
1.2.1. Persistent failures still produce one error per configured poll cycle;
there is no cross-cycle circuit breaker or jitter beyond the client's bounded
per-request retry behavior.

## Transport boundary

This pull runner intentionally does not replace transports with stronger native
delivery semantics:

- Azure Event Hubs Ingester: Azure Firewall, Application Gateway/WAF, Storage,
  Key Vault, AKS, Azure SQL, API Management, Front Door/WAF, supported Virtual
  Network Flow Logs, and other Azure Monitor resource-log categories.
- Office 365 Ingester: Microsoft 365 Management Activity content.
- Simple Relay: Defender for IoT sensor CEF, LEEF, or syslog.
- File Follower: explicit append-only NDJSON exports.

These transports may be deployed beside the pull runner. Duplicates are only
acceptable when deliberately enabled and labeled during validation.

## Validation

Run these commands from the repository root with the Go version specified in
`go.mod`. The formatting check should print no paths. The build produces a
`hosted-runner` executable in the current directory.

```bash
gofmt -l hosted/plugins/microsoft hosted/plugins/builder.go hosted/plugins/config.go
go test -tags upstream_registration ./hosted/... -count=1
go test ./hosted/... -count=1
go test -race -tags upstream_registration ./hosted/... -count=1
go vet -tags upstream_registration ./hosted/...
go build -trimpath -o hosted-runner ./hosted/runner
```

The Hosted Runner builds for macOS amd64/arm64 and Linux amd64. Windows is not
currently supported because the pinned Gravwell Hosted Runner Bolt storage
implementation directly uses Unix `syscall.Flock`.

Microsoft is registered by `hosted/plugins` and included in `hosted/runner`. A site
that needs a Microsoft-only lifecycle deploys that binary with a configuration
containing only `[Microsoft "name"]` stanzas and separate cache/state/log
mounts. There is no second, divergent Microsoft Hosted Runner source tree.

The example uses all-zero dummy identifiers by design and must fail production
configuration validation until each UUID and Microsoft identifier is replaced.
Do not put real tenant, client, subscription, backend, or secret values in this
repository.
