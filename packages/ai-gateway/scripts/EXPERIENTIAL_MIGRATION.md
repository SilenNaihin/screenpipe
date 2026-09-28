# Experiential migration preparation

Status: preparation and synthetic probes only. These scripts stay outside the
Worker import graph. They do not change the production provider factory,
Cloudflare routing, secrets, webhooks, deployment configuration or traffic.

```text
Before: customer -> existing gateway/auth/limits -> existing providers
After:  customer -> existing gateway/auth/limits -> existing providers
        operator -> read-only preflight -> Experiential account
        operator -> explicitly approved synthetic probe -> Experiential
```

## Offline inventory

From `packages/ai-gateway`:

```sh
bun scripts/export-migration-inventory.ts > migration-inventory.json
```

Imports the current repository model/plan policy. Destination mappings and live
allowances remain null. Source credit constants are not a live rule export. Run
from a clean reviewed checkout; the report records the current Git revision.
Keep generated reports private when enriched with account or production facts.

## Read-only preflight

Load the test credential into `EXPLABS_API_KEY` from your secret manager inside
the calling process. Never put it in CLI arguments, source, logs or a ticket.
Do not use your Cloudflare or upstream provider credential for this request.

```sh
bun scripts/experiential-preflight.ts > experiential-preflight.json
```

Only GETs the documented API host. Compares exact authenticated model IDs and
reads prompt capture and upstream policy. Refuses redirects, redacts upstream
errors, and never tries another key or host. Exit 1 means inspection failed;
exit 2 means a report was produced with migration gates still unresolved. Catalog
access cannot mark the account production ready.

## Optional synthetic probes

Require spending approval, a test-only credential, billing activation, disabled
prompt capture and no-training routing. Preflight runs again each time. Each
invocation sends exactly one synthetic POST, at most 64 output tokens, with no
retries or fallback. Review the model price before approving: token bounds are
not a dollar guarantee. Reconcile uncertain requests before retrying.

```sh
bun scripts/experiential-smoke.ts --allow-spend gpt-5.4-nano chat
bun scripts/experiential-smoke.ts --allow-spend gpt-5.4-nano stream
bun scripts/experiential-smoke.ts --allow-spend gpt-5.4-nano tools
bun scripts/experiential-smoke.ts --allow-spend gpt-5.4-nano json-schema
bun scripts/experiential-smoke.ts --allow-spend gpt-6-astra responses
bun scripts/experiential-smoke.ts --allow-spend claude-sonnet-5 messages
```

Run only approved cases. These are examples, not a batch command. An unavailable
model fails without substitution. Responses keeps Astra stateless with low
reasoning and standard tier. Confidential/custom, classifier, Auto and background
rescue IDs are excluded. Streaming requires content, successful finish, usage and
DONE. Tool/schema checks inspect actual arguments or JSON, not just HTTP status.
The output omits prompts, completion content, headers and secrets. Inline cost is
optional and never proves settlement or customer allowance accounting.

These probes test the vendor wire contract, not the deployed Screenpipe Worker
or a completed routing adapter. Images, cache accounting, provider-native stream
translation and multi-customer enforcement still need end-to-end proof.

## Inactive adapter and local tests

`lib/experiential-adapter.ts` is outside the Worker import graph. Its default
mode refuses all activity; only an explicit `isolated-test` caller can prepare
one request. It does not select a production provider, synchronize entitlements,
or activate a cutover. The existing Bun tests are its consumers:

```sh
bun test src/test/experiential-migration.unit.test.ts src/test/experiential-adapter.unit.test.ts src/test/experiential-sdk-parity.unit.test.ts
```

The SDK tests exercise the installed OpenAI and Anthropic serializers and stream
parsers using injected synthetic transports. They cover tools, images, cache
usage, structured-output serialization, and Astra Responses-to-Chat translation.
Astra stays Low, Standard (`service_tier: "default"` on the wire), and stateless
(`store: false`). Anthropic's existing schema support remains system-prompt
instructions, not a claim of native constrained decoding.

The adapter requires server-verified Screenpipe authentication and a trusted
server registry binding the actor to a customer identity, customer key, source
plan, and expected destination plan key/version/assignment revision. The
provisioning identity and management credential stay separate from inference.
It performs these actual reads before preparing a request:

- Management key: organization, disabled `capture_prompt_content`, no-training
  policy, exact catalog slug, and `/api/models/{slug}` canonical UUID.
- Customer key: `/api/whoami` with both identity fields matching the registry
  and `is_provisioning: false`, customer-visible model listing, and its own
  `customer-plan` assignment matching the pinned tuple with no pending change.

Older servers without identity proof fail closed. Missing models, alternate
model guesses, stale assignments, shared management/customer keys, and privacy
failures never dispatch. Gateway identities are artifact IDs; organization and
canonical model IDs are UUIDs. The adapter sends at most one inference POST per
prepared connection, refuses replay, strips caller/provider/Cloudflare auth
headers, and defaults to 64 output tokens. An operator may explicitly select
128 for an isolated probe; higher or missing request bounds are refused. Neither
bound is a dollar guarantee or permission to spend.

The opt-in HTTP integration test uses a separately provisioned, synthetic-only
local platform stack. It reads a private JSON fixture from the environment and
refuses non-loopback URLs, any model other than `gpt-5.4-mini`, or a fixture not
marked `syntheticUpstream: true`:

```sh
SCREENPIPE_EXPERIENTIAL_LOCAL_FIXTURE=/absolute/private-fixture.json \
  bun test src/test/experiential-adapter.integration.test.ts --timeout=30000
```

The fixture contains `controlBaseUrl`, `gatewayBaseUrl`, `managementKey`, `orgId`,
`provisioningIdentityId`, `canonicalModelUuid`, `model`, `syntheticUpstream`, and
two `customers`. Each customer has `userId`, `identityId`, `customerKey`,
`accountPlan`, and `expectedPlan: { plan_key, plan_version, revision }`. Obtain
these values from the isolated stack's actual control API; do not fabricate
policy readbacks. Keep the fixture outside source control. Without it, the test
is skipped. It exercises one JSON and one streaming request under distinct
customer keys, but neither provisions policy nor proves real-provider behavior.

The provider fixes preserve OpenAI's upstream nonstream `finish_reason` and
normalize Anthropic nonstream tool-call IDs/types and terminal reasons into Chat
format. Regression tests cover a second tool turn. These fixes do not change
routing or imply deployment. Encrypted Tinfoil/EHBP, ordinary custom GLM,
Vertex, classifier, audio, and background rescue routes remain unchanged.

## Gates before implementing production selection

1. Pin deployed behavior and read actual Cloudflare limits, reset windows,
   settled usage, reservations and late/pending exposure.
2. Verify native enforcement across the serving fleet. The documented disjoint
   model groups and UTC-month allowances do not alone preserve shared-total plus
   frontier caps or other windows. Do not invent or approximate commercial limits.
3. Bind customer keys to authenticated identities and approved plans before
   issuance. Keep management credentials separate. Test two synthetic identities,
   rotation, forbidden models, concurrent holds, plan transitions and scoped
   reporting. Request labels authorize nothing.
4. Verify exact model mappings, rates, privacy, cache accounting and retries.
   Keep encrypted Tinfoil/EHBP, custom GLM, Vertex Messages, classifier, audio
   and background rescue separate without their own compatibility proof.
5. Reconcile usage carry-in and rollback across both gateways. A fresh destination
   balance must not reset usage or grant duplicate allowance.
6. Then review a separate routing change and stage with synthetic users.
   Production promotion still requires the owner's explicit signal.

Do not use an unrestricted org key as a temporary customer route. These scripts
never mint keys, assign plans, upload BYOK, change settings, purchase, deploy,
connect subscription webhooks or revoke existing gateway credentials.

Official contracts checked September 27, 2026:

- [Account and identities](https://platform.experientiallabs.ai/docs/account-api)
- [Usage and allowances](https://platform.experientiallabs.ai/docs/cost-api)
- [Privacy](https://platform.experientiallabs.ai/docs/data-controls)
- [Wire reference](https://platform.experientiallabs.ai/llms.txt)
