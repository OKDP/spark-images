# Spark 4.0 source patches and dependency overrides

This directory contains the source patches and pombump configuration used to
customize upstream Spark 4.0 before building the image. Source patches change
code or module dependencies; pombump pins dependency versions in the root Maven
POM. Together, they provide image-specific runtime adjustments, JSON logging
support, and a reproducible dependency baseline.

The currently configured target in
[pre-build-patch-pombump.yml](../../.build/pre-build-patch-pombump.yml) is
**Spark 4.0.5, Java 17, Python 3.11, and Hadoop 3.4.1**. The build action matches
all four values before processing the source. The files in this directory are not
all applied to every Spark 4.0 release. The build applies only the patches listed
for the matching configuration; for Spark 4.0.5, see [Source patches](#source-patches).

## Source patches

For Spark 4.0.5, the build applies the patches listed in
[pre-build-patch-pombump.yml](../../.build/pre-build-patch-pombump.yml), in the
order shown below, before pombump. The table follows that order.

| File | Applied for 4.0.5 | Introduced change | Goal |
|---|---|---|---|
| [aws-sdk-v2-slim.patch](aws-sdk-v2-slim.patch) | Yes | In `hadoop-cloud/pom.xml`, excludes `software.amazon.awssdk:bundle` and declares `s3-transfer-manager`, `sts`, `kms`, `apache-client`, and `netty-nio-client` at `${aws.java.sdk.v2.version}`. | Replaces the full shaded AWS SDK v2 bundle with the service and HTTP client modules that are used, to slim the distribution. |
| [bump-packages.patch](bump-packages.patch) | Yes | Adds `org.apache.logging.log4j:log4j-layout-template-json` to `core/pom.xml`, without an explicit version. | Makes Log4j's JSON Template Layout available to Spark core; the version comes from Maven dependency management. This adds the library but does not configure a JSON logging layout. |
| [entrypoint.patch](entrypoint.patch) | Yes | Changes the Kubernetes container entrypoint from `/usr/bin/tini` to `/sbin/tini`, preserving `-s -- "${CMD[@]}"`. | Aligns process startup with the image's `tini` location while preserving signal forwarding and child-process reaping. |
| [lz4.patch](lz4.patch) | Yes | Changes the lz4-java coordinates from `org.lz4:lz4-java` to `at.yawk.lz4:lz4-java` in `core/pom.xml` and `connector/kafka-0-10-assembly/pom.xml`, and changes the root `pom.xml` managed version from `1.8.0` to `1.11.1`. | Moves to the `at.yawk.lz4` artifact at version 1.11.1. |
| [path-update-scripts.patch](path-update-scripts.patch) | Yes | In `sbin/spark-daemon.sh`, calls `/usr/bin/nohup` and `/usr/bin/ps` by absolute path instead of resolving them from `PATH`. | Makes the daemon script use fixed binary locations. |
| [spark_submit.patch](spark_submit.patch) | Yes | Wraps glob resolution for `args.jars`, `args.files`, `args.pyFiles`, and `args.archives` in `!isKubernetesCluster && sparkConf.getBoolean("spark.kubernetes.resolveGlobPathsInSubmit", true)`. | Avoids submit-side glob resolution in Kubernetes cluster mode and allows it to be disabled in other modes. |

The patches are applied with `patch -p1` in the order listed above. This README does not
record whether they apply cleanly to upstream tag `v4.0.5`; the build's patch step is the check.

The patches listed as applied have not been checked against the upstream source
in this directory; the build's `patch -p1` run is the check for that.

### Resource glob resolution behavior

The submission patch changes when Spark expands resource path patterns:

- **Kubernetes cluster mode:** skips this glob-resolution block, regardless of
  the configuration value.
- **Other modes:** preserves upstream glob resolution by default.
- Setting `spark.kubernetes.resolveGlobPathsInSubmit=false` also skips the block
  in other modes, despite the Kubernetes-prefixed setting name.

This setting only controls the four assignments above; it does not disable all
resource handling or guarantee that a later stage will expand a given pattern.

## pombump configuration

The build action currently uses **pombump v0.0.16** to rewrite the root `pom.xml`
after the source patches. The YAML files have complementary roles:

- [pombump-properties.yaml](pombump-properties.yaml) supplies named Maven
  property values, shared by dependencies that reference those properties.
- [pombump-deps.yaml](pombump-deps.yaml) supplies explicit dependency coordinates
  and versions, including references back to the shared properties.

These are requested target versions, not necessarily upgrades relative to every
upstream tag. A value may already match the source. Setting a property also does
not prove that a dependency consumes it or is included in the final image.

### Property targets

| Maven property | Target value |
|---|---|
| `log4j.version` | `2.26.1` |
| `netty.version` | `4.1.137.Final` |
| `aws.java.sdk.v2.version` | `2.41.34` |
| `bouncycastle.version` | `1.85` |
| `guava.version` | `33.8.0-jre` |
| `jersey.version` | `3.0.18` |
| `zookeeper.version` | `3.9.7` |
| `commons-lang3.version` | `3.18.0` |
| `hadoop.version` | `3.4.3` |
| `fasterxml.jackson.version` | `2.18.11` (first entry) and `2.18.9` (second entry) |
| `jetty.version` | `11.0.26` |
| `fasterxml.jackson.databind.version` | `2.18.11` |
| `codehaus.jackson.version` | `1.9.14.jdk17-redhat-00001` |

`fasterxml.jackson.version` is defined twice in
[pombump-properties.yaml](pombump-properties.yaml) with different values. Resolve
this in the YAML before relying on either value.

### Dependency targets

[pombump-deps.yaml](pombump-deps.yaml) has two entries:

| Dependency (`groupId:artifactId`) | Target version | Scope | Type |
|---|---|---|---|
| `io.airlift:aircompressor` | `2.0.3` | `import` | `jar` |
| `io.vertx:vertx-core` | `4.5.36` | `import` | `pom` |

The YAML files are the source of truth; update these tables when changing their values.

## Processing and validation

The existing [patch-pombump action](../../.github/actions/patch-pombump/action.yml):

1. Selects the matching build configuration and checks out Spark tag `v4.0.5`.
2. Applies the patches listed for that configuration with `patch -p1`
   (for 4.0.5: `aws-sdk-v2-slim.patch`, `bump-packages.patch`, `entrypoint.patch`,
   `lz4.patch`, `path-update-scripts.patch`, and `spark_submit.patch`; a listed file that is missing is skipped).
3. Runs pombump against the patched root POM using both YAML files.
4. Replaces the root POM with the generated output and copies the modified
   source tree into the image build context.

Successful patch application and pombump execution are configuration checks,
not proof of dependency compatibility or vulnerability remediation. Review the
generated XML and pombump log, then validate the Spark build and affected runtime
behavior. Use the required OSPC SQAAS gates for security verification.

pombump re-serializes the POM, so textual diffs can include substantial formatting
and element-order changes in addition to the requested version changes.
