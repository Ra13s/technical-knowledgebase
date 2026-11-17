# Test AWS SDK integrations against isolated local AWS Emulator instances

## What it is

AWS Emulator (released November 11, 2026) is a locally running, Smithy-model-derived AWS API implementation for developer and CI tests. Unlike a pure mock of SDK method calls, it accepts real SigV4-signed SDK requests over HTTPS, maintains resource state in memory, and returns documented validation errors for implemented operations. Unsupported operations return HTTP 501; **service coverage does not mean behavior fidelity**.

Use this as a **fast integration-test stage**, not a substitute for a real AWS staging account, IAM evaluation or security testing. The Rust source is open under CC0 (excluding separately licensed vendor crates).

## Use when

- Java/Spring or other applications use SQS, S3, DynamoDB or several AWS services;
- AI coding agents need repeatable cloud-adjacent tests without AWS account credentials;
- CI requires isolated, low-latency test environments per worker;
- you want to diagnose API request-shape and state-transition bugs before costly staging tests.

Avoid treating emulator success as proof of IAM, networking, regional, concurrency or durability behavior in AWS production.

## Quick-start for Java/Spring teams

Use the documented Debian/RPM package on Linux or the vendor's Docker instructions for Windows/macOS. The emulator defaults to HTTPS at \`127.0.0.1:9297\`, generating **per-start** emulator-only credentials, SDK config and a self-signed trust store under \`~/.aws/emulator/9297/\`.

For a local Linux process after installation:

~~~bash
# In a dedicated test shell with NO genuine AWS keys present:
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE
export EMULATOR_TELEMETRY=off
aws-emulator --telemetry off
~~~

From another shell (or test startup script):

~~~bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE
export AWS_ENDPOINT_URL=https://127.0.0.1:9297
export AWS_SHARED_CREDENTIALS_FILE="$HOME/.aws/emulator/9297/emulator_credentials"
export AWS_CONFIG_FILE="$HOME/.aws/emulator/9297/config"
export AWS_CA_BUNDLE="$HOME/.aws/emulator/9297/ca.pem"
export AWS_EC2_METADATA_DISABLED=true
export AWS_DISABLE_HOST_PREFIX_INJECTION=true
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
# Java SDK 2.x needs a Java trust store; AWS_CA_BUNDLE alone isn't enough:
export JAVA_TOOL_OPTIONS="-Djavax.net.ssl.trustStore=$HOME/.aws/emulator/9297/truststore.p12 -Djavax.net.ssl.trustStoreType=PKCS12"

curl -fsS --cacert "$HOME/.aws/emulator/9297/ca.pem" \
  https://127.0.0.1:9297/__admin/health
aws sts get-caller-identity
# Expected Account == 000000000000, Arn ends in user/emulator.
~~~

When running Maven/Gradle, **prefer setting the trust-store JVM args on the test process only** rather than globally: this trust store replaces the JDK default and will otherwise break unrelated HTTPS calls. The SDK for Java 2.x may require explicit \`SdkAdvancedClientOption.DISABLE_HOST_PREFIX_INJECTION\` for operations that rewrite the endpoint hostname. For S3 use IP \`127.0.0.1\`, not \`localhost\`, to avoid virtual-hosted bucket hostname routing.

Test with the real AWS SDK, e.g. a Spring integration test that creates a queue, sends and receives one message, deletes it, and asserts the response. Use a fresh emulator process/port per test worker; AWS Emulator state is in memory only and is lost on exit.

~~~java
// Schematic AWS SDK v2 test client; trust-store and credentials
// are supplied to the test process as shown above.
SqsClient sqs = SqsClient.builder()
    .region(Region.US_EAST_1)
    .endpointOverride(URI.create("https://127.0.0.1:9297"))
    .build();
// Exercise CreateQueue -> SendMessage -> ReceiveMessage -> DeleteMessage
// and verify business-level effects, not only HTTP status.
~~~

### AI coding-agent configuration

AWS CLI v2.35.0+ provides the \`aws-emulator\` skill:

~~~sh
aws agent-toolkit add-skill --skill-name aws-emulator --region us-east-1
~~~

A coding agent can use it to identify which AWS operations the project actually calls and compare them to available fidelity tests; it must not gain access to genuine AWS credentials simply for local testing.

### Fidelity gate

1. List all SDK operations used by the service and the stateful flows they compose.
2. Read the **pinned emulator release's** fidelity scores and the public evaluation-suite \`manifest.jsonl\`; score existence of a tested operation separately from successful behavior.
3. Run integration tests locally, with the network sandbox denying outbound AWS endpoints.
4. For critical workflows or missing coverage, run the *same test* in an isolated AWS test account, compare semantic results and classify differences. Destroy disposable AWS resources.
5. Keep a real AWS staging run for IAM, regions, real service limits, timeouts, race behavior, TLS and security.

If a method returns 501, report \`unsupported\`, not a product bug or “passed mock”. If an operation has no oracle-backed fidelity test, mark it \`unverified\`, not compatible.

## Failure modes and constraints

- **No real IAM semantics or account isolation:** the emulator's generated credentials are for its local server only. Never disable SigV4 authentication in shared or untrusted environments.
- **Accidental real-cloud calls:** missing endpoint variables can route tests to AWS; enforce an explicit identity check (\`000000000000\`) and block real AWS egress/credentials.
- **TLS trust store:** rotating emulator certs and a Java-specific PKCS12 truststore; replacing JVM defaults may break unrelated HTTPS.
- **State lifetime / concurrency:** in-memory state disappears at restart. Use separate processes/ports or embedded mode (the latter is Rust-only, and compilation is substantial).
- **Regional differences:** AWS Emulator is not region-dependent. A test passing in one emulated region says nothing about multi-region AWS behavior.
- **Operational coverage:** 200+ service APIs can be present while many behaviors are incomplete. Compare per-operation fidelity, not marketing service counts.
- **Maturity:** first release; pin a tested version, monitor issue regressions and do not run production traffic through it. Telemetry is on by default unless explicitly disabled.

## Experiment

In one Spring integration module, compare (A) Mockito AWS SDK stubs, (B) AWS Emulator with real SDK v2 and SigV4, and (C) an isolated AWS test account, for SQS and DynamoDB create/read/update/delete flows. Intentionally inject invalid schemas, conditional-write conflicts and missing resources. Record unsupported calls, differences from real AWS, test runtime, cost, isolation leaks and seeded bug detection. Keep only validated emulator behaviors as CI merge gates; route unverified/high-risk behaviors to staging.

## Sources

- AWS release announcement (2026-11-11): https://aws.amazon.com/about-aws/whats-new/2026/11/aws-emulator-now-available/
- Source and README: https://github.com/aws/aws-emulator
- SDK configuration and Java trust-store caveats: https://docs.aws.amazon.com/emulator/latest/userguide/point-sdk-to-emulator.html
- Check fidelity coverage: https://docs.aws.amazon.com/emulator/latest/userguide/check-project.html

## Related

- [Reference-oracle autonomous agent factory](../ai/reference-oracle-autonomous-agent-factory.md)
- [Microcks Testcontainers contract tests](../testing/microcks-testcontainers-contract-tests.md)
- [Record/replay model calls](../testing/ai-model-vcr-record-replay-tests.md)
