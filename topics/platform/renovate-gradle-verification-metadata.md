# Renovate + Gradle dependency verification metadata

## What it is

Gradle dependency verification pins trusted checksums/signatures in `gradle/verification-metadata.xml`. When Renovate upgrades a dependency, the new artifact is intentionally not trusted yet, so CI can fail until verification metadata is updated.

Use Renovate `postUpgradeTasks` to regenerate the verification files in the same dependency-update PR.

## Use when

Use this when all of these are true:

- the project uses Gradle dependency verification;
- Renovate opens dependency-update PRs;
- those PRs fail because the updated artifact/version has no verification metadata yet;
- you control or self-host Renovate sufficiently to allow a tightly-scoped post-upgrade command.

## How to use it

Repository `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "postUpgradeTasks": {
    "commands": [
      "./gradlew --write-verification-metadata pgp,sha256 --refresh-keys --export-keys dependencies"
    ],
    "fileFilters": [
      "gradle/verification-metadata.xml",
      "gradle/verification-keyring.gpg",
      "gradle/verification-keyring.keys"
    ],
    "installTools": {
      "java": {}
    }
  }
}
```

Self-hosted Renovate blocks post-upgrade commands by default. The bot administrator must explicitly allow the exact command with `allowedCommands`; keep that allowlist narrow.

Gradle's simpler manual form is:

```bash
./gradlew --write-verification-metadata sha256,pgp
```

Gradle incrementally updates `verification-metadata.xml`; generated checksums still require review.

## Why it is useful

Without this, dependency automation and dependency verification fight each other: every legitimate version bump creates a second manual PR just to refresh verification metadata.

This recipe keeps the security mechanism enabled while putting the metadata update in the same PR as the dependency change, where the new artifact/signature/checksum can be reviewed together.

## Caveats / when not to use

- **Generating verification metadata is not verification of trust.** Gradle explicitly warns that bootstrapping trusts the artifacts currently downloaded. Review critical dependency changes and generated metadata.
- `postUpgradeTasks` executes code in Renovate's environment. Restrict `allowedCommands`, permissions, environment access and repository trust.
- Avoid enabling shell execution unless it is genuinely required; Renovate warns that shell semantics weaken command allowlisting.
- Hosted Renovate installations may restrict which post-upgrade commands are available.
- Commit only the keyring format you actually use; Gradle can export binary and ASCII-armored keyrings.

## Version / compatibility

The recipe relies on current Gradle dependency-verification CLI options and Renovate `postUpgradeTasks`. Check both products' current docs when adopting it because security-related execution controls can evolve.

## Sources

- https://blog.frankel.ch/solving-gradle-metadata-renovate-integration/
- https://docs.gradle.org/current/userguide/dependency_verification.html
- https://docs.renovatebot.com/configuration-options/#postupgradetasks
- https://docs.renovatebot.com/self-hosted-configuration/#allowedcommands
- https://docs.renovatebot.com/security-and-permissions/
- Discovery: https://www.baeldung.com/java-weekly-660
