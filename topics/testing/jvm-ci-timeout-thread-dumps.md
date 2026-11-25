# Capture JVM thread dumps before CI kills hung tests

## Use when

Maven/Gradle tests intermittently hang on CI and the only evidence is a timeout. Make diagnostic capture part of the test lifecycle, not a rerun that may no longer reproduce the same state.

## Maven Surefire / Failsafe 3.6.0+

The fork-timeout extension is invoked while the timed-out fork is still alive. Pin a compatible plugin version (example 3.6.1) in the project POM. Requires a full JDK with jstack and a resolvable fork PID; use Java 9+ (prefer the project's supported JDK). Configure Failsafe separately if integration tests are the problem.

~~~xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.6.1</version>
  <configuration>
    <forkedProcessTimeoutInSeconds>600</forkedProcessTimeoutInSeconds>
    <forkedProcessTimeoutExtensionContext>
      <jstack.enabled>true</jstack.enabled>
    </forkedProcessTimeoutExtensionContext>
  </configuration>
</plugin>
~~~

Alternatively, enable the built-in capture with the Maven system property:

~~~bash
./mvnw -B test -Dsurefire.timeout=600 -Dsurefire.timeout.jstack.enabled=true
~~~

A fork timeout is a lifetime budget, not an individual-test timeout. Calibrate per suite; a healthy reused fork must fit in the budget. The report is normally under the configured reports directory as surefire-timeout-jstack-*.txt. Capture is best effort; missing jstack or attach failure must not turn the failed test into success. The extension SPI is available for project-specific diagnostics if truly necessary; keep callback execution bounded.

### CI budget and artifacts

Use an inner test/fork timeout shorter than the CI step limit, which is shorter than the job deadline. Example GitHub Actions excerpt inside a job with a configured full JDK, checkout, and job timeout of 30 minutes:

~~~yaml
- name: Verify
  run: ./mvnw -B verify
  timeout-minutes: 25
- name: Upload hang diagnostics
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: jvm-timeout-diagnostics
    path: |
      **/target/surefire-reports/*jstack*.txt
      **/target/failsafe-reports/*jstack*.txt
    if-no-files-found: ignore
    retention-days: 7
~~~

Pin the upload action according to repo policy. Ensure the job's deadline also accommodates checkout/build and artifact upload. Restrict artifacts to the repository's approved access and retention; thread names, stack paths and lock identifiers may be sensitive. Do not send dumps to external analysis services by default.

### Gradle or older test runner: watchdog

For Linux CI, start a supervised watchdog alongside the test command that samples matching test JVMs before the outer timeout. Example collection fragment (run inside the isolated job under the same OS user as test JVMs):

~~~bash
mkdir -p thread-dumps
jcmd -l | awk '/GradleWorkerMain|surefirebooter/ {print $1}' |
while read -r pid; do
  timeout 15s jcmd "$pid" Thread.print -l > "thread-dumps/$pid.txt" || true
done
~~~

Schedule the watchdog with enough time for capture/upload and terminate it when the build exits; preserve original test exit status. Validate actual JVM command names on your build. For non-forked Maven tests, target the Maven PID only after validating process identity. Avoid collecting unrelated processes on shared hosts. On recent Java, virtual-thread-heavy apps may also need Thread.dump_to_file; check supported diagnostic commands with jcmd PID help.

A failure/cancellation step can sometimes attach after the test step is killed, but this depends on runner process cleanup and is not a guarantee. Prefer inner timeout hook or pre-deadline watchdog. Hard VM termination, OOM kill, or attach restrictions can defeat both.

## Interpret and verify

Look for lock cycles, blocked test thread, latch/future waits, pool starvation, network calls without deadlines; compare repeated samples to distinguish a slow test from a stalled one. Preserve CI run/commit identity with the dump. Do not let an agent automatically patch synchronization from a single stack trace.

Experiment in a disposable test job: create an intentionally blocked latch and a two-lock deadlock with short timeouts, then confirm (1) the test fails, (2) dumps identify expected stacks, (3) upload completes, (4) ordinary passing jobs do not leave a watchdog running, (5) no unrelated processes or secrets are captured. Try both Surefire and Failsafe/Gradle as appropriate. This is a proposed local check, not a test executed during intake.

## Evidence and sources

- Felipe Maschio (11 Nov 2026) reproduced surviving forked JVMs after cancellation on hosted Linux/macOS runners; this is a practitioner experiment, not a durable runner contract: https://foojay.io/today/how-i-got-a-thread-dump-from-a-timed-out-github-actions-test/
- Apache Maven extension SPI and built-in activation: https://maven.apache.org/surefire-archives/surefire-LATEST/maven-surefire-plugin/examples/timeout-extension.html
- Failsafe parameters: https://maven.apache.org/components/surefire-archives/surefire-LATEST/maven-failsafe-plugin/integration-test-mojo.html
- GitHub cancellation behavior: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-cancellation

## Related

- [Aggregate required CI checks](../platform/github-actions-aggregate-required-check.md)
- [Review AI-generated Maven changes](../ai/review-ai-generated-maven-build-changes.md)
