# E6: How can a storage implementation be supplied at boot?

- **Script:** `validation/experiments/E6-bootstrap-seam.sh`
- **Spike source:** `validation/experiments/E6/src`
- **Raw evidence:** `raw/E6/`

**Spike:** `E6LoggingConfidentialStore` is a `ConfidentialStore` that keeps keys in memory and logs each access. It is registered via `META-INF/services/jenkins.security.ConfidentialStore`, which `ConfidentialStore.get()` discovers with `ServiceLoader.load(ConfidentialStore.class, ConfidentialStore.class.getClassLoader())` (`core/src/main/java/jenkins/security/ConfidentialStore.java:82`).

| Variant | Result |
|---|---|
| (a) jar on Winstone `--commonLibFolder` | **Boot failure:** `NoClassDefFoundError: jenkins/security/ConfidentialStore` at `ConfidentialStore.get(ConfidentialStore.java:83)` |
| (b) jar injected into the war's `WEB-INF/lib` | **Works.** Instantiated by `WebAppClassLoader{Jenkins v2.580.1}`. Every `ConfidentialKey` goes through it (`crumbSalt`, `User.DIRNAMES`, …), and **no `secrets/` directory is created** |

## Why
- Winstone's common-lib `URLClassLoader` is the **parent** of the webapp class loader (`winstone/src/main/java/winstone/Launcher.java:137-158`). Classes it loads cannot see `jenkins-core` in `WEB-INF/lib`, so the service class's superclass doesn't resolve.
- `ConfidentialStore` declares `abstract SecureRandom secureRandom()` as **package-private** (`ConfidentialStore.java:61`). An implementation must therefore sit in package `jenkins.security` *and* be loaded by the same class loader as core. The spike only works because of both.
- `secret.key` (`Jenkins.java:931`) was still written in variant (b). It doesn't go through `ConfidentialStore`.

## Implications for Target 1
- Implementations of a pre-plugin storage SPI must be on the **webapp class path**. Two ways to get there:
  - Build a custom war: add a jar to `WEB-INF/lib`, as the Custom WAR Packager / a Dockerfile layer would.
  - Add a supported "early classpath" option to core or Winstone.

  Plugins and `--commonLibFolder` won't work.
- Core changes needed:
  - Make `ConfidentialStore` implementable outside its package: promote `secureRandom()` to `protected`, with a default. The TODO at line 60 already suggests this.
  - Route `secret.key` through the same store.
