# Roll out hybrid post-quantum TLS key exchange in Java 27

## What it is

**JDK 27** adds TLS 1.3 hybrid key-exchange groups in the SunJSSE provider (JEP 527). The preferred default `X25519MLKEM768` combines classical X25519 with post-quantum ML-KEM-768; it is offered **alongside classical groups** so connections can still negotiate with peers that do not implement the hybrid option.

For applications using standard `javax.net.ssl`/JSSE implementations this is primarily a **runtime interoperability, observability and rollout task**, not a custom crypto implementation. Both sides must support the hybrid group for it to be negotiated; merely upgrading one side is not evidence of post-quantum protection on a connection.

## Use when

- upgrading a Java service's client/server runtime to JDK 27;
- planning a post-quantum key-exchange compatibility canary for Spring Boot HTTP, messaging or database connections that use JSSE;
- inspecting TLS handshake failures after a Java or reverse-proxy upgrade;
- evaluating custom `jdk.tls.namedGroups` or `SSLParameters` overrides that might silently remove new defaults.

## What ships

SunJSSE in JDK 27 supports:

| TLS named group | Components |
|---|---|
| `X25519MLKEM768` (default-preferred) | X25519 + ML-KEM-768 |
| `SecP256r1MLKEM768` | P-256 + ML-KEM-768 |
| `SecP384r1MLKEM1024` | P-384 + ML-KEM-1024 |

These are **TLS 1.3 key-exchange** groups, not post-quantum certificate/signature replacement. Classical certificates and authentication still need their own security lifecycle. This protects against the **harvest-now, decrypt-later** risk of *key agreement*, conditional on the security assumptions of the hybrid construction.

## Implementable rollout recipe

1. Inventory actual TLS termination for a sample of service-to-service, outbound HTTP, gRPC, JDBC and external ingress paths. Identify whether JSSE, Netty/tcnative/OpenSSL, a sidecar or a load balancer does the handshake. A JDK upgrade changes only the paths using the upgraded Java provider.
2. Pin the exact JDK distribution/update and inspect explicit named-group policies. The JDK 27 SunJSSE default already includes hybrid plus classical groups.
3. In a **test environment**, log the negotiated handshake with:
   
   ```sh
   java -Djavax.net.debug=ssl:handshake -jar application.jar
   ```

   Use this only in a controlled environment; debug output may include sensitive peer metadata. Inspect the group negotiated on each **end-to-end TLS leg**, not just advertised support.
4. For compatibility testing, use the system property on a **canary only** to set an explicit ordered group list:

   ```sh
   java -Djdk.tls.namedGroups=X25519MLKEM768,x25519,secp256r1 -jar application.jar
   ```

   Alternatively set groups per socket using `SSLParameters.setNamedGroups(...)`:

   ```java
   SSLSocket socket = (SSLSocket) SSLContext.getDefault()
           .getSocketFactory().createSocket();
   SSLParameters p = socket.getSSLParameters();
   p.setNamedGroups(new String[] {
       "X25519MLKEM768", "x25519", "secp256r1"
   });
   socket.setSSLParameters(p);
   ```

   The last snippet configures parameters; a real TLS connection still requires setting hostname/port, enabling hostname verification as appropriate, and connecting/handshaking. **Do not copy it as a complete production client.**
5. Exercise a matrix: JDK 27 client ↔ JDK 27 server; JDK 27 ↔ older/non-hybrid peer; and an outbound connection through every TLS proxy. Test ALPN, mTLS, TLS session resumption and large handshake messages with network middleboxes.
6. Inspect handshake failures, p95/p99 connect latency, TLS CPU overhead and negotiated group distribution. Compare fallback rates before enabling wider rollout.
7. Retain an explicit classical-group rollback that satisfies current organizational crypto policy; never disable TLS peer certificate/hostname checks to fix a group negotiation problem.

## Verification gates

```text
JDK/JSSE path confirmed?
 -> default/hybrid group advertised?
 -> peer actually negotiated hybrid?
 -> certificate validation and application protocol passed?
 -> fallback tested against older peers?
 -> handshake payload / latency / CPU acceptable?
 -> canary health + rollback ready?
```

Assert **negotiated** group on representative connections. Unit tests that only inspect a list of supported names are not evidence that production connections use hybrid TLS. If a client and peer negotiate classical `x25519`, the connection remains a valid classical TLS 1.3 connection, not a hybrid one.

## Compatibility and caveats

- **JDK 27 availability:** this implementation is generally available in JDK 27. JDK 21/25 backports and vendor updates vary; confirm the exact deployed build rather than assuming any given older runtime supports JEP 527.
- **Defaults can be overridden:** custom named-group lists can suppress hybrid support without an error. A corporate TLS policy may also deliberately constrain named groups.
- **Non-JSSE engines:** a native TLS stack used by an HTTP framework, agent or proxy may have different capabilities.
- **Larger key shares:** hybrid negotiation increases TLS handshake size; validate proxies, fragmentation, buffers and latency on real routes.
- **No PQ authentication:** hybrid KEX does not replace RSA/ECDSA certificate signatures or secure stored traffic data.
- **Mixed ecosystems:** support may differ per provider, load balancer and JDK vendor. Don't force hybrid-only for the first rollout.

## Experiment

Build a JUnit/Testcontainers integration matrix with a simple JSSE TLS 1.3 endpoint in JDK 27 and an older peer, capturing negotiated-group debug evidence as a redacted test artifact. Then canary one Spring service with outbound `RestClient`/JDK HttpClient traffic and measure observed hybrid fraction, fallbacks, handshake errors and latencies. Confirm which client factory actually uses JSSE.

## Sources

- OpenJDK JEP 527: https://openjdk.org/jeps/527
- Java 27 JSSE reference: https://docs.oracle.com/en/java/javase/27/security/java-secure-socket-extension-jsse-reference-guide.html
- Inside Java, JDK 27 security changes (2026-09-16): https://inside.java/2026/09/16/jdk27-security-enhancements/
- Inside Java, exact named-group examples: https://inside.java/2026/02/17/tls-post-quantum-hybrid-key-exchange/

## Related

- [Java 26 HTTP/3 with the standard HttpClient](java-26-http3-httpclient.md)
- [JSpecify + NullAway null-safety](../spring/jspecify-null-safety.md)
