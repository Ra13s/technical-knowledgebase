# Java 26 HTTP/3 with `java.net.http.HttpClient`

## What it is

JDK 26 adds HTTP/3 support to the standard `java.net.http.HttpClient` through `HttpClient.Version.HTTP_3` and the request option `HttpOption.H3_DISCOVERY`.

HTTP/3 is **opt-in** in the JDK client. The default preferred protocol remains HTTP/2, and an HTTP/3-enabled request can still negotiate or fall back differently depending on its discovery mode, endpoint and proxy configuration.

## Use when

Evaluate this when a Java service calls HTTPS endpoints that support HTTP/3 and you want to measure whether QUIC improves latency or connection behavior for your real network path.

Good candidates include:

- latency-sensitive service-to-service or edge calls;
- clients on lossy or mobile networks;
- services where repeated connection establishment is significant;
- compatibility tests for an API/CDN that is introducing HTTP/3.

Do not move production traffic to JDK 26 only for HTTP/3 without an upgrade/support plan: JDK 26 is a non-LTS release.

## Basic opt-in

Enable HTTP/3 as the client's preferred version:

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

var client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_3)
    .build();

var request = HttpRequest.newBuilder(URI.create("https://example.com/api"))
    .GET()
    .build();

var response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.version());
```

Always inspect `response.version()` in tests/telemetry if HTTP/3 usage itself matters. Setting a preferred version is not proof that the exchange used it.

## Choose discovery behavior explicitly when it matters

`HttpOption.H3_DISCOVERY` accepts `HttpOption.Http3DiscoveryMode`.

### `ANY` — normal fallback-friendly opt-in

When HTTP/3 is selected and no `H3_DISCOVERY` hint is supplied, the JDK built-in client uses `ANY`.

Use this as the default experiment when you want the client implementation to choose how to establish the exchange while still allowing fallback.

```java
var client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_3)
    .build();
```

### `ALT_SVC` — wait for server advertisement

Use this when you want HTTP/3 discovery through HTTP Alternative Services rather than aggressively requiring a direct H3 attempt.

```java
import java.net.http.HttpOption;

var request = HttpRequest.newBuilder(URI.create("https://example.com/api"))
    .version(HttpClient.Version.HTTP_3)
    .setOption(
        HttpOption.H3_DISCOVERY,
        HttpOption.Http3DiscoveryMode.ALT_SVC)
    .GET()
    .build();
```

This is useful when HTTP/3 availability is expected to be learned from the server and graceful HTTP/2/1.1 operation is normal.

### `HTTP_3_URI_ONLY` — fail rather than silently downgrade

Use this for compatibility tests or workloads where the request must go directly to the URI authority using HTTP/3.

```java
var request = HttpRequest.newBuilder(URI.create("https://example.com/api"))
    .version(HttpClient.Version.HTTP_3)
    .setOption(
        HttpOption.H3_DISCOVERY,
        HttpOption.Http3DiscoveryMode.HTTP_3_URI_ONLY)
    .GET()
    .build();
```

If HTTP/3 cannot be used, failure is preferable to hiding the condition behind fallback.

## Decision rule

```text
want H3 opportunistically
    -> HTTP_3 + default ANY

want H3 only after server advertises it
    -> HTTP_3 + ALT_SVC

want to prove direct H3 works, no silent fallback
    -> HTTP_3 + HTTP_3_URI_ONLY
```

Use the request-level preferred version when only selected calls should try HTTP/3. Use the client-level version when it is the normal preference for that client instance.

## Important constraints

### HTTPS only

The JDK built-in implementation never sends a non-`https` request over HTTP/3.

### Proxy behavior

The built-in JDK 26 client does not support HTTP/3 through a selected proxy.

- With normal discovery/fallback, the request can downgrade to HTTP/2 or HTTP/1.1 and the H3 discovery hint is ignored.
- With `HTTP_3_URI_ONLY`, the request fails instead of downgrading.

This matters in enterprise environments where local tests bypass a proxy but production traffic does not.

### Unsupported client configuration can fail

A client implementation can throw `UnsupportedProtocolVersionException` when it cannot support HTTP/3, for example because its configured SSL context/parameters are incompatible with the implementation's HTTP/3 requirements.

## Rollout / test recipe

1. verify the target endpoint actually advertises/supports HTTP/3;
2. add request telemetry for `response.version()`, latency, error/fallback rate and connection establishment;
3. test the real production network path, including proxies and firewalls;
4. compare HTTP/2 baseline vs `ANY` rather than assuming QUIC is faster;
5. use `HTTP_3_URI_ONLY` in a focused compatibility test to detect silent fallback;
6. keep a JDK upgrade plan because JDK 26 is non-LTS.

For example, an integration assertion can intentionally require H3:

```java
var response = client.send(h3OnlyRequest, HttpResponse.BodyHandlers.discarding());
assert response.version() == HttpClient.Version.HTTP_3;
```

Do not turn that assertion into a general production availability rule unless the service contract truly requires HTTP/3.

## Why it is useful

Before JDK 26, using HTTP/3 from Java generally required another HTTP stack/library. JDK 26 makes the protocol available behind the standard `HttpClient` API and gives applications an explicit choice between opportunistic discovery and hard H3-only verification.

The practical value is not "HTTP/3 is always faster". It is that protocol rollout and fallback behavior can now be tested with a small, standard-library change and measured against the real network.

## Caveats

- JDK 26 is non-LTS; confirm runtime support policy before production adoption.
- HTTP/3 is not selected by default.
- QUIC/UDP behavior depends on the real network path; benchmark from representative environments.
- Proxy use can make a local H3 test misleading.
- `H3_DISCOVERY` has no effect unless HTTP/3 is selected as the client or request preferred version.
- Prefer one long-lived `HttpClient` over constructing a client per request so connections can be reused.

## Sources

- Inside Java, *HTTP Client Updates in Java 26* (2026-03-04): https://inside.java/2026/03/04/jdk-26-http-client/
- Oracle Java SE 26 `HttpClient`: https://docs.oracle.com/en/java/javase/26/docs/api/java.net.http/java/net/http/HttpClient.html
- Oracle Java SE 26 `HttpClient.Version`: https://docs.oracle.com/en/java/javase/26/docs/api/java.net.http/java/net/http/HttpClient.Version.html
- Oracle Java SE 26 `HttpOption`: https://docs.oracle.com/en/java/javase/26/docs/api/java.net.http/java/net/http/HttpOption.html

## Related

- [Switching over evolving sealed APIs](evolving-sealed-api-switches.md)
