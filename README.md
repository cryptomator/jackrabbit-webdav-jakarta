# jackrabbit-webdav-jakarta

`org.apache.jackrabbit:jackrabbit-webdav`, mechanically transformed from `javax.servlet` to `jakarta.servlet` and modularized.

This is an **interim artifact**: Apache Jackrabbit has not yet released jakarta-compatible artifacts. Upstream migration is tracked in [JCR-4892](https://issues.apache.org/jira/browse/JCR-4892); as of May 2026 the plan is to switch the next stable branch (2.24/3.0) entirely to `jakarta.servlet` + Java 17. Once such a release exists, consumers should switch back to the official artifact — the package names are unchanged (`org.apache.jackrabbit.webdav.*`), so this is a drop-in.

## How it works

There is no real source code in this repository. The build resolves the upstream `jackrabbit-webdav` jar (and its `sources` jar) and runs the [Eclipse Transformer](https://github.com/eclipse-transformer/transformer) Maven plugin's `jakartaDefaults` rules over it, producing this project's main and sources artifacts. The POM re-declares upstream's runtime dependencies, with `javax.servlet-api` replaced by `jakarta.servlet-api`.

Upstream ships no module descriptor. After the transform, the [ModiTect](https://github.com/moditect/moditect) Maven plugin compiles [`src/main/moditect/module-info.txt`](src/main/moditect/module-info.txt) and injects it into the jar root — the transformed classes are Java 11 bytecode, so no multi-release layout is needed.

## Java Platform Module System

The jar is a proper named module:

```java
requires org.apache.jackrabbit.webdav;
```

The official jackrabbit-webdav library is **not modularized**. Once this happens, this dependency can be replaced.

The module is named after its root exported package, per JPMS convention, so that migrating to an official jakarta-capable `org.apache.jackrabbit:jackrabbit-webdav` needs no change to consumer module descriptors. All `org.apache.jackrabbit.webdav.*` packages are exported. `jakarta.servlet`, `java.xml` and the two httpcomponents modules are `requires transitive` because they appear in exported API signatures; `org.slf4j` is not.

### Server-only consumers

`httpclient` and `httpcore` are reachable only from `org.apache.jackrabbit.webdav.client.methods` and `util.LinkHeaderFieldParser`. They are declared `requires transitive static`, so a consumer that only uses the server side can drop them from its runtime — but a static `requires` is still **mandatory at compile time**, so javac needs them on the module path anyway. `provided` scope does both:

```xml
<dependency>
	<groupId>org.cryptomator</groupId>
	<artifactId>jackrabbit-webdav-jakarta</artifactId>
	<version>${jackrabbit.version}</version>
	<exclusions>
		<exclusion>
			<groupId>org.apache.httpcomponents</groupId>
			<artifactId>httpclient</artifactId>
		</exclusion>
		<exclusion>
			<groupId>org.apache.httpcomponents</groupId>
			<artifactId>httpcore</artifactId>
		</exclusion>
		<exclusion>
			<!-- commons logging is only used by httpclient -->
			<groupId>org.slf4j</groupId>
			<artifactId>jcl-over-slf4j</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<!-- excluded above, but a static requires still has to resolve at compile time -->
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpclient</artifactId>
	<version>4.5.14</version>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpcore</artifactId>
	<version>4.4.16</version>
	<scope>provided</scope>
</dependency>
```

Consumers that *do* use `client.methods` simply omit all of the above and take both as normal transitive dependencies.

## Versioning

The project version mirrors the upstream Jackrabbit version being transformed (`${jackrabbit.version}` in the POM — keep both in sync). Repackaging-only fixes would be published as e.g. `2.22.4-1`.

## Known caveats

- **Servlet 6 removed methods:** `WebdavRequestImpl`/`WebdavResponseImpl` still contain delegating overrides of methods removed in Jakarta Servlet 6.0 (`getRealPath`, `isRequestedSessionIdFromUrl`, `encodeUrl`, `encodeRedirectUrl`, `setStatus(int, String)`). These are dead code under a 6.0 API jar and throw `NoSuchMethodError` if ever invoked. The WebDAV framework itself never calls them.
- **`upgrade()` generic signature:** the transformer misses the type-variable bound in the generic *signature attribute* of `WebdavRequestImpl.upgrade(Class)` — it still names `javax.servlet.http.HttpUpgradeHandler`. This only matters if consumer code compiles a direct call to `WebdavRequestImpl.upgrade(...)` (nothing does; the erased method descriptor is correctly transformed).
- **OSGi metadata:** the transformed manifest imports `jakarta.servlet;version="[5.0,6)"`, i.e. it does not admit a Servlet 6.x bundle. Irrelevant outside OSGi.
- **`jlink` only works for the server part:** `httpclient` and `httpcore` are automatic modules and thus cannot be linked into a runtime image. Because both are `requires static`, jlink does not resolve them and server-only consumers link fine (see [Server-only consumers](#server-only-consumers)). Consumers of `client.methods` still cannot use jlink — inherited from upstream's dependency choices, not from the transform.
- **The sources jar has no `module-info.java`:** ModiTect rewrites only the binary jar, so the sources artifact is missing that one file relative to the main artifact.

## License

The repackaged code is Apache Jackrabbit, licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0). The jar retains upstream's `META-INF/LICENSE` and `META-INF/NOTICE`.
