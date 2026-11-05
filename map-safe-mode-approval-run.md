# Nitrosend Safe Mode & Mail Action Protocol (MAP 0.3) Approval Run Report

- **Exact installation, tools, and runtime versions:**
  - OS & Kernel: Ubuntu 24.04.4 LTS (`Linux 7.0.0-31-generic x86_64`).
  - Node.js & npm: `v22.14.0` (`node --version`), `npm 10.9.2` (installed in `/tmp/node22` because Ubuntu 24.04's system `Node.js v18.19.1` lacks JSON import attributes `with { type: 'json' }` required by `mailschema@0.3.2`).
  - Packages installed in `/tmp/map_142`:
    ```sh
    npm init -y
    npm install mailschema@0.3.2 @nitrosend/sdk@0.12.0
    curl -sLO https://mailschema.org/artifacts/map-0.3/contracts/campaign-send-approval-0.1.json
    ```
- **Exact commands and API calls run:**
  - **Step 0 (Prerequisite send receipt via `@nitrosend/sdk@0.12.0`):** Provisioned account `14257` (`POST https://api.nitrosend.com/v1/signup`), configured Brand Kit (`PATCH /v1/my/brands/cdcd84893f1e244caf548913` with physical mailing address), activated hosted sender `hello@jmquero92.nitrosend.net` (`POST /v1/my/brands/cdcd84893f1e244caf548913/prepare_sending`), created API key `11376` (`POST /v1/my/api_keys`), and sent one email to `proof@dogfood.nitrosend.net` via `ns.messages.send({ channel: "email", to: "proof@dogfood.nitrosend.net", subject: "ElegantFermi SDK verification send", html: "<p>Verification send from ElegantFermi Labs using @nitrosend/sdk 0.12.0.</p>" }, "sdk-step0-20261105-01")` (message `id: 5380237`, send receipt `https://runx.ai/r/26afec63672371a2d1719e4c6749fb72b4e444da14896b8cada218a922d08565`).
  - **Step 1 & 2 (Enable Safe Mode):** Enabled Safe Mode on the account at `2026-11-05T18:20:50Z`:
    ```sh
    curl -X PATCH https://api.nitrosend.com/v1/my/account       -H "Authorization: Bearer <owner-jwt>"       -H "Content-Type: application/json"       -d '{"safe_mode_enabled":true}'
    ```
  - **Step 3 (Agent campaign creation and scheduled send held by Safe Mode):** Authenticated with the API key (`Bearer nskey_live_<redacted>`), created contact `4240867` (`POST /v1/my/contacts`), created list `13273` (`"SafeMode Verification List"`, `POST /v1/my/lists`), created campaign `16239` (`"Autumn Infrastructure Digest"`, `POST /v1/my/campaigns`), updated its template (`version: 2`, no merge tags) and list audience (`PATCH /v1/my/campaigns/16239`), and scheduled delivery 30 minutes ahead (`2026-11-05T18:50:51Z`):
    ```sh
    curl -X POST https://api.nitrosend.com/v1/my/campaigns/16239/send       -H "Authorization: Bearer nskey_live_<redacted>"       -H "Content-Type: application/json"       -d '{"deliver_at":"2026-11-05T18:50:51Z","expected_campaign_updated_at":"2026-11-05T18:20:51.624Z","template_if_version":2}'
    ```
    Nitrosend held the campaign with `delivery.status: "awaiting_approval"` (`wait_reason: "manual_approval"`) and emailed `"Review send request: Autumn Infrastructure Digest"`.
  - **Step 4 (Extract MAP 0.3 description from raw `.eml`):** Saved the arrival email as `approval.eml` (`32,542` bytes) and extracted the `application/ld+json; profile="https://mailschema.org/profiles/map/0.3"` MIME part:
    ```sh
    npx mailschema@0.3.2 message approval.eml > description.json
    ```
    Standard error output from `mailschema message`:
    ```text
    Valid MAP 0.3 description urn:uuid:9d21a5be-69ce-54a9-9bf5-a0e6ddfab45e
    sha-256:d46247733dc6f8d4c424eb9b6710a7a3a3d071170a9a039dd6b1c09b2cdcdc85
    ```
- **Minutes from turning Safe Mode on to decision:** Approximately **14 minutes** from enabling Safe Mode (`2026-11-05T18:20:50Z`) and receiving the held campaign email (`2026-11-05T18:20:51Z`) to recording the approval decision at `2026-11-05T18:34:59.666Z` (receipt issued at `2026-11-05T18:34Z`).
- **Validate command and exact output:**
  - Command:
    ```sh
    npx mailschema@0.3.2 description description.json --contract campaign-send-approval-0.1.json
    ```
  - Exact output:
    ```text
    Valid MAP 0.3 description urn:uuid:9d21a5be-69ce-54a9-9bf5-a0e6ddfab45e
    sha-256:d46247733dc6f8d4c424eb9b6710a7a3a3d071170a9a039dd6b1c09b2cdcdc85
    ```
- **How `terms.version` was recomputed and exact value obtained:**
  - Parsed `description.json`, cloned the JSON object, deleted `clone.terms.version` (`delete clone.terms.version`), and passed the modified object to `digest(clone)` from `mailschema@0.3.2` (which computes `sha-256:` over the RFC 8785 JCS canonicalization):
    ```sh
    node --input-type=module -e '
    import fs from "fs";
    import { digest } from "mailschema";
    const desc = JSON.parse(fs.readFileSync("description.json", "utf8"));
    const origVersion = desc.terms.version;
    const clone = structuredClone(desc);
    delete clone.terms.version;
    const recomputed = digest(clone);
    console.log("terms.id:", desc.terms.id);
    console.log("orig terms.version:", origVersion);
    console.log("recomputed digest: ", recomputed);
    console.log("match:", origVersion === recomputed);
    '
    ```
  - Exact output and value:
    ```text
    terms.id: https://api.nitrosend.com/v1/my/map/approvals/820
    orig terms.version: sha-256:faa1fb88bceae6ea6e815461751312ac671fb9c339dd6c9b95bab31795b12c3f
    recomputed digest:  sha-256:faa1fb88bceae6ea6e815461751312ac671fb9c339dd6c9b95bab31795b12c3f
    match: true
    ```
- **Decision made and why:**
  - **Decision:** **`approve`** (`POST https://api.nitrosend.com/v1/my/map/approvals/820` with `{"decision":"approve","version":"sha-256:faa1fb88bceae6ea6e815461751312ac671fb9c339dd6c9b95bab31795b12c3f"}`, decided at `2026-11-05T18:34:59.666Z`).
  - **Why:** The extracted MAP 0.3 description passed core and `campaign-send-approval` `0.1` contract validation (`contractDigest: sha-256:de3c88a0712adb1d0c6e82f437b825c83520d8e1c44c7b032a2cfcfe5ce55731`), the recomputed RFC 8785 digest without `terms.version` matched `terms.version` (`sha-256:faa1fb88bceae6ea6e815461751312ac671fb9c339dd6c9b95bab31795b12c3f`), and the proposal's `details` (`sender.mailbox: "hello@jmquero92.nitrosend.net"`, `audience.count: 1`, `content.revision: "2"`, `schedule.at: "2026-11-05T18:50:51Z"`) matched the exact campaign snapshot we authored.
- **Step that slowed us most:**
  - Upgrading the local Node.js runtime from Ubuntu 24.04's default `Node.js v18.19.1` to `Node.js v22.14.0` after `npx mailschema@0.3.2` threw `SyntaxError: Unexpected token 'with'` on `import coreSchema from '../core.schema.json' with { type: 'json' };` in `dist/core/bundled.js`, and ensuring the prerequisite send to `proof@dogfood.nitrosend.net` used `@nitrosend/sdk@0.12.0` (`ns.messages.send`) rather than raw `curl`/`urllib` against `POST /v1/my/messages` so the client identified itself and generated the prerequisite send receipt (`https://runx.ai/r/26afec63672371a2d1719e4c6749fb72b4e444da14896b8cada218a922d08565`).
- **Defect / documentation gap and exact reproduction:**
  - **Issue 1 (`mailschema@0.3.2` CLI `bin/mailschema.js` rejects `/dev/stdin` and process substitution `<(...)` and checks `CONTRACT_MAX_BYTES` instead of `DESCRIPTION_MAX_BYTES`):** In `bin/mailschema.js` (lines 32–37 and 56–60), `read(path, maxBytes = CONTRACT_MAX_BYTES)` runs `const info = await stat(path); if (!info.isFile() || info.size > maxBytes) throw new Error(...)`. Because `info.isFile()` is `false` for FIFOs and character devices (`/dev/stdin`, `/dev/fd/*`), piping `mailschema message approval.eml` into `mailschema description /dev/stdin` (or `<(npx mailschema@0.3.2 message approval.eml)`) fails with `Expected a JSON file of at most 262144 bytes.`. Additionally, `command === 'description'` calls `read(path)` with the default `CONTRACT_MAX_BYTES` (`262144`) instead of `DESCRIPTION_MAX_BYTES` (`65536` exported in `dist/core/artifacts.js`).
  - **Issue 2 (Nitrosend Safe Mode API returns `delivery_approval: null` on `POST /v1/my/campaigns/{id}/send` and keeps `can_decide: true` after approval):** When `POST /v1/my/campaigns/16239/send` transitions the campaign to `delivery.status: "awaiting_approval"`, the top-level `delivery_approval` field in the response is `null` (and `openapi.json` has no `GET /v1/my/map/approvals` list endpoint), so the approval ID (`820`) is only discoverable from the email's `terms.id` (`https://api.nitrosend.com/v1/my/map/approvals/820`). Then, when `POST /v1/my/map/approvals/820` succeeds with `"state": "approved"` and `"decided_at": "2026-11-05T18:34:59.666Z"`, the returned `SendApproval` object still reports `"can_decide": true`.
  - **Exact reproduction:**
    ```sh
    # 1. Piping mailschema message stdout into mailschema description via /dev/stdin:
    npx mailschema@0.3.2 message approval.eml 2>/dev/null | npx mailschema@0.3.2 description /dev/stdin --contract campaign-send-approval-0.1.json
    # Output (exit code 1):
    # Expected a JSON file of at most 262144 bytes.

    # 2. Inspecting POST /v1/my/map/approvals/820 response after approving:
    # Returns {"id": 820, "state": "approved", "decided_at": "2026-11-05T18:34:59.666Z", "can_decide": true, ...}
    ```
- **One change we would make:**
  - In `mailschema` (`bin/mailschema.js`), allow `mailschema description -` (or `/dev/stdin`) by reading from `process.stdin` with `DESCRIPTION_MAX_BYTES` (`65536`) so operators and agents can pipe `mailschema message approval.eml | mailschema description - --contract campaign-send-approval-0.1.json` in a single pipeline; and in Nitrosend's `/v1/my/map/approvals/{id}` serializer, set `can_decide: false` once `state` is `"approved"` or `"declined"`.
