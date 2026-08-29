# Spring Boot outbound SSRF mitigation with `InetAddressFilter`

## What it is

Spring Boot 4.1 adds `org.springframework.boot.http.client.InetAddressFilter`, an address-level filter that can reject outbound HTTP destinations after DNS resolution and before a connection is opened.

## Use when

Use it when application code can make outbound HTTP calls to URLs or hosts influenced by users, configuration, plugins, agents, webhooks, import jobs or other semi-trusted input.

Typical examples:

- URL preview/fetch endpoints
- webhook validation or callbacks
- agent tools that fetch arbitrary URLs
- document/import pipelines that dereference remote resources
- generic proxy/integration services

The reusable rule is: **validate the resolved destination address, not only the hostname string**.

## Global policy for auto-configured Spring HTTP clients

For applications that should normally call only public Internet addresses, expose a filter bean:

```java
import org.springframework.boot.http.client.InetAddressFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
class OutboundHttpSecurityConfig {

    @Bean
    InetAddressFilter outboundAddressFilter() {
        return InetAddressFilter.externalAddresses();
    }
}
```

Spring Boot applies an `InetAddressFilter` bean to its auto-configured HTTP client builders, so clients created from those builders inherit the policy.

`externalAddresses()` excludes loopback/private/special-purpose destinations. This blocks common SSRF targets such as localhost and private network ranges.

## Per-client policy

When one integration needs a different network policy, apply a filter to that client’s `HttpClientSettings` instead of weakening the application-wide rule:

```java
InetAddressFilter onlyExternal = InetAddressFilter.externalAddresses();

HttpClientSettings settings = HttpClientSettings.defaults()
    .withInetAddressFilter(onlyExternal);

ClientHttpRequestFactory factory = ClientHttpRequestFactoryBuilder
    .jdk()
    .build(settings);

RestClient client = RestClient.builder()
    .requestFactory(factory)
    .build();
```

## Compose allowlists / denylists

The API supports IPv4/IPv6 addresses and CIDR blocks:

```java
InetAddressFilter partnerNetwork = InetAddressFilter
    .of("203.0.113.0/24")
    .andNot("203.0.113.42");
```

Useful factories/combinators include:

- `externalAddresses()`
- `internalAddresses()`
- `specialPurpose()`
- `multicast()`
- `of(...)`
- `and(...)`, `andNot(...)`, `or(...)`, `negate()`

Prefer a narrow allowlist when the integration has a stable destination range. Use `externalAddresses()` when arbitrary public URLs are intentionally supported.

## Test the boundary

Add an integration test proving loopback/private destinations are rejected:

```java
assertThatThrownBy(() -> client.get()
        .uri("http://127.0.0.1:8080/admin")
        .retrieve()
        .toBodilessEntity())
    .isInstanceOf(FilteredHostException.class);
```

Also test any explicitly allowed internal or partner ranges so a future policy edit does not silently broaden or break access.

## Why it is useful

- The check is based on the resolved `InetAddress`, reducing hostname-only SSRF bypasses.
- The rule can be centralized for Spring-managed clients.
- Per-client policies remain possible for integrations with different trust boundaries.
- The policy is visible and testable in application code rather than being scattered across interceptors.

## Caveats / when not to use

- This is defense in depth, not a replacement for network egress controls, DNS security, proxy policy or cloud metadata protections.
- A global `externalAddresses()` rule will intentionally break clients that must call private service addresses; give those clients an explicit narrower policy rather than disabling filtering globally.
- Ensure application code actually builds clients from the configured Spring Boot builders/settings. A separately constructed third-party HTTP client will not inherit this policy automatically.
- Address filtering does not validate HTTP response content, redirects, authentication, or application-level authorization.

## Version / compatibility

`InetAddressFilter` is available since Spring Boot **4.1.0**. Verified against Spring Boot 4.1.1 documentation.

## Sources

- Spring Boot REST client reference: https://docs.spring.io/spring-boot/reference/io/rest-client.html
- `InetAddressFilter` API: https://docs.spring.io/spring-boot/api/java/org/springframework/boot/http/client/InetAddressFilter.html
- Discovery/example: https://www.baeldung.com/spring-boot-http-client-ssrf-mitigation-inetaddressfilter
