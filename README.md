# jackrabbit-webdav-jakarta

`org.apache.jackrabbit:jackrabbit-webdav`, mechanically transformed from `javax.servlet` to `jakarta.servlet` at build time using the [Eclipse Transformer](https://github.com/eclipse-transformer/transformer) Maven plugin.

This is an **interim artifact**: Apache Jackrabbit has not yet released jakarta-compatible artifacts. Upstream migration is tracked in [JCR-4892](https://issues.apache.org/jira/browse/JCR-4892); as of May 2026 the plan is to switch the next stable branch (2.24/3.0) entirely to `jakarta.servlet` + Java 17. Once such a release exists, consumers should switch back to the official artifact — the package names are unchanged (`org.apache.jackrabbit.webdav.*`), so this is a drop-in.

## How it works

There is no source code in this repository. The build resolves the upstream `jackrabbit-webdav` jar (and its `sources` jar) and runs the Eclipse Transformer's `jakartaDefaults` rules over it, producing this project's main and sources artifacts. The POM re-declares upstream's runtime dependencies, with `javax.servlet-api` replaced by `jakarta.servlet-api`.

## Versioning

The project version mirrors the upstream Jackrabbit version being transformed (`${jackrabbit.version}` in the POM — keep both in sync). Repackaging-only fixes would be published as e.g. `2.22.3-1`.

## Known caveats

- **Servlet 6 removed methods:** `WebdavRequestImpl`/`WebdavResponseImpl` still contain delegating overrides of methods removed in Jakarta Servlet 6.0 (`getRealPath`, `isRequestedSessionIdFromUrl`, `encodeUrl`, `encodeRedirectUrl`, `setStatus(int, String)`). These are dead code under a 6.0 API jar and throw `NoSuchMethodError` if ever invoked. The WebDAV framework itself never calls them.
- **`upgrade()` generic signature:** the transformer misses the type-variable bound in the generic *signature attribute* of `WebdavRequestImpl.upgrade(Class)` — it still names `javax.servlet.http.HttpUpgradeHandler`. This only matters if consumer code compiles a direct call to `WebdavRequestImpl.upgrade(...)` (nothing does; the erased method descriptor is correctly transformed).
- **OSGi metadata:** the transformed manifest imports `jakarta.servlet;version="[5.0,6)"`, i.e. it does not admit a Servlet 6.x bundle. Irrelevant outside OSGi.

## License

The repackaged code is Apache Jackrabbit, licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0). The jar retains upstream's `META-INF/LICENSE` and `META-INF/NOTICE`.
