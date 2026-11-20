# fcp-audit-performance-test-suite

A JMeter based performance test suite for [fcp-audit](https://github.com/DEFRA/fcp-audit), run on the CDP Platform.

fcp-audit's primary function is to ingest audit messages that FCP services publish to SNS, and then store them so they can be queried. This suite measures:

1. **Ingestion**: how quickly fcp-audit consumes a known volume of audit messages from its queue and stores them
2. **API read performance**: how the audit API responds under sustained concurrent load

- [Licence](#licence)
  - [About the licence](#about-the-licence)

## Build

Test suites are built automatically by the [.github/workflows/publish.yml](.github/workflows/publish.yml) action whenever a change are committed to the `main` branch.
A successful build results in a Docker container that is capable of running your tests on the CDP Platform and publishing the results to the CDP Portal.

## Run

The performance test suites are designed to be run from the CDP Portal.
The CDP Platform runs test suites in much the same way it runs any other service, it takes a docker image and runs it as an ECS task, automatically provisioning infrastructure as required.

## Local Testing

The `fcp-audit-performance-test-suite` can be ran locally via Docker compose.

You will first need to run the service you wish to test against locally.

Once this is complete, you can trigger a local test run:
```
./test.sh
```

Test reports are saved locally and can be viewed in `reports` directory.

## Tests

The scenario in [scenarios/test.jmx](scenarios/test.jmx) runs two tests, one after the other.

### 1. Ingestion test (setUp thread group)

This test measures how quickly fcp-audit ingests audit messages. [fcp-audit-publisher-stub](https://github.com/DEFRA/fcp-audit-publisher-stub) generates the load by publishing messages to the same SNS topic that real FCP services use. The time measured runs until those messages can be read back from the audit API.

```
publisher stub ──▶ SNS ──▶ SQS ──▶ fcp-audit consumer ──▶ MongoDB ──▶ GET /api/v1/audit/summary
```

A single thread:

1. **Audit summary (baseline)**: reads the current audit total from `GET /api/v1/audit/summary`
2. **Publish audit messages**: asks the stub to publish `INGEST_REPETITIONS` repetitions of its scenarios. The stub responds `202` straight away and publishes in the background
3. **Audit summary (poll)**: polls the summary every `INGEST_POLL_INTERVAL_MS` until the total reaches the baseline plus `INGEST_REPETITIONS × INGEST_AUDIT_EVENTS_PER_REPETITION`, or `INGEST_TIMEOUT_SECONDS` passes
4. **Verify audit messages ingested**: fails if ingestion did not complete. Its response message gives the number of messages ingested, the time taken and the ingestion rate in events per second. The message also appears in the JMeter log and the CSV report

In the HTML report, the **Ingest audit messages** row is the end-to-end ingestion time, from the publish request until the last message can be read from the audit API.

When you read the results, keep in mind:

- **The measurement is end to end.** It covers the stub's publishing as well as fcp-audit's consumption. The stub publishes one message at a time, so the result may be limited by the stub rather than fcp-audit. Watch the fcp-audit queue's `ApproximateNumberOfMessagesVisible` metric during the test. A growing backlog means fcp-audit is the limit; a queue that stays near empty means the stub is
- **An idle consumer starts slowly.** When fcp-audit's queue has been empty, it backs off for up to 32 seconds between polls. For small volumes this start-up delay can dominate the result
- **The poll interval limits precision.** The end time can be up to `INGEST_POLL_INTERVAL_MS` late
- **The count leaves out fcp-audit's own API audit events.** fcp-audit publishes an audit event (application `Audit Service`) for each of its own API calls, so these are excluded. Anything else that publishes audit events during the test will make ingestion look faster than it is

The test is skipped when `DISABLE_EVENTS=true`.

### 2. API read load (main thread group)

This starts only after the ingestion test has finished. 75 threads ramp up over 30 seconds and then run for 10 minutes, each repeatedly calling `GET /health` and `GET /api/v1/audit`.

The API load also puts fcp-audit under some ingestion load. Each `GET /api/v1/audit` call publishes an API audit event, which fcp-audit then ingests, so reads and writes happen together.

### Ingestion test configuration

| Variable | Default | Description |
| --- | --- | --- |
| `DISABLE_EVENTS` | `false` | Skip the ingestion test |
| `INGEST_REPETITIONS` | `5000` | Repetitions passed to the stub's simulate endpoint |
| `INGEST_AUDIT_EVENTS_PER_REPETITION` | `2` | Audit records stored per repetition. This must match the stub's scenarios: it publishes `event`, `auditEvent` and `socEvent`, and only the first two contain an `audit` section that fcp-audit stores |
| `INGEST_TIMEOUT_SECONDS` | `600` | How long to wait for ingestion before failing the test |
| `INGEST_POLL_INTERVAL_MS` | `2000` | How often to poll the audit summary |

## Authentication

The test suite supports optional Microsoft Entra (Azure AD) authentication using OAuth 2.0 client credentials flow. This allows testing of authenticated APIs without manual token management.

### How It Works

1. **Token Acquisition**: During the Setup Thread Group, a single access token is requested from Microsoft Entra (`login.microsoftonline.com`)
2. **Token Sharing**: The token is extracted and stored in the `${access_token}` variable, shared across all test threads
3. **Authorization Header**: When enabled, each request to the Messages API automatically includes `Authorization: Bearer ${access_token}`

### Configuration

Authentication is controlled via environment variables in the `.env` file:

```env
# Enable/disable authentication
AUTH_ENABLED=true

# Microsoft Entra credentials
TENANT_ID=your-azure-tenant-id
CLIENT_ID=your-app-client-id
CLIENT_SECRET=your-app-secret
```

### Token Lifecycle

- Tokens are acquired **once** during test setup
- The token is shared across all threads
- The token expires after 60 minutes, they remain valid for the complete test run

### Disabling Authentication

To test unauthenticated endpoints or run tests in environments without authentication:

```env
AUTH_ENABLED=false
```

When disabled, the test suite skips token acquisition and does not add Authorization headers to requests.

## Synthetic CRN/SBI check

A pre-commit hook and the pull request workflow use [fcp-sfd-core](https://github.com/DEFRA/fcp-sfd-core)'s `sfd-id-check` to block 10-digit (CRN) or 9-digit (SBI) numbers outside the synthetic ranges. Use synthetic-range values in test scenarios, properties and examples.

The hook needs Node.js and npm. `.npmrc` sets `ignore-scripts=true`, so install the git hooks yourself once after `npm install`:

```bash
npm install
npm run setup:husky
```

To run the check on staged files by hand:

```bash
npm run check:synthetic-ids
```

## Licence

THIS INFORMATION IS LICENSED UNDER THE CONDITIONS OF THE OPEN GOVERNMENT LICENCE found at:

<http://www.nationalarchives.gov.uk/doc/open-government-licence/version/3>

The following attribution statement MUST be cited in your products and applications when using this information.

> Contains public sector information licensed under the Open Government licence v3

### About the licence

The Open Government Licence (OGL) was developed by the Controller of Her Majesty's Stationery Office (HMSO) to enable
information providers in the public sector to license the use and re-use of their information under a common open
licence.

It is designed to encourage use and re-use of information freely and flexibly, with only a few conditions.

