# Spark 4.0 source patches and dependency overrides

This directory contains the source patches and pombump configuration used to
customize upstream Spark 4.0 before building the image. Source patches change
code or module dependencies; pombump pins dependency versions in the root Maven
POM. Together, they provide image-specific runtime adjustments, JSON logging
support, and a reproducible dependency baseline.

The currently configured target in
[pre-build-patch-pombump.yml](../../.build/pre-build-patch-pombump.yml) is
**Spark 4.0.5, Java 17, Python 3.11, and Hadoop 3.4.1**. The build action matches
all four values before processing the source. These files are not automatically
applied to every Spark 4.0 release.

## Source patches

The patches are applied in the following order, before pombump:

| File | Introduced change | Goal |
|---|---|---|
| [bump-packages.patch](bump-packages.patch) | Adds `org.apache.logging.log4j:log4j-layout-template-json` to `core/pom.xml`, without an explicit version. | Makes Log4j's JSON Template Layout available to Spark core; the version comes from Maven dependency management. This adds the library but does not configure a JSON logging layout. |
| [entrypoint.patch](entrypoint.patch) | Changes the Kubernetes container entrypoint from `/usr/bin/tini` to `/sbin/tini`, preserving `-s -- "${CMD[@]}"`. | Aligns process startup with the image's `tini` location while preserving signal forwarding and child-process reaping. |
| [spark_submit.patch](spark_submit.patch) | Wraps glob resolution for `args.jars`, `args.files`, `args.pyFiles`, and `args.archives` in `!isKubernetesCluster && sparkConf.getBoolean("spark.kubernetes.resolveGlobPathsInSubmit", true)`. | Avoids submit-side glob resolution in Kubernetes cluster mode and allows it to be disabled in other modes. |

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
| `hadoop.version` | `3.4.1` |
| `ivy.version` | `2.5.3` |
| `log4j.version` | `2.24.3` |
| `slf4j.version` | `2.0.16` |
| `avro.version` | `1.12.0` |
| `snappy.version` | `1.1.10.7` |
| `commons-compress.version` | `1.27.1` |
| `commons-io.version` | `2.18.0` |
| `commons-codec.version` | `1.17.2` |
| `guava.version` | `33.4.0-jre` |
| `netty.version` | `4.1.118.Final` |
| `zookeeper.version` | `3.9.3` |
| `gcs-connector.version` | `hadoop3-2.2.26` |
| `aws.java.sdk.version` | `1.11.655` |
| `fasterxml.jackson.version` | `2.18.6` |
| `fasterxml.jackson.databind.version` | `2.18.6` |
| `aws.kinesis.producer.version` | `0.12.8` |
| `aws.kinesis.client.version` | `1.12.0` |
| `jetty.version` | `11.0.24` |

### Dependency targets

All five entries in [pombump-deps.yaml](pombump-deps.yaml) specify `scope: import`
and `type: jar`.

| Dependency (`groupId:artifactId`) | Target version |
|---|---|
| `com.google.code.gson:gson` | `2.11.0` |
| `org.apache.commons:commons-compress` | `${commons-compress.version}` (`1.27.1`) |
| `io.airlift:aircompressor` | `2.0.3` |
| `commons-io:commons-io` | `${commons-io.version}` (`2.18.0`) |
| `commons-codec:commons-codec` | `${commons-codec.version}` (`1.17.2`) |

Keep the `${...}` references aligned with the property targets. The YAML files
are the source of truth; update these tables when changing their values.

## Processing and validation

The existing [patch-pombump action](../../.github/actions/patch-pombump/action.yml):

1. Selects the matching build configuration and checks out Spark tag `v4.0.5`.
2. Applies the three source patches above with `patch -p1`.
3. Runs pombump against the patched root POM using both YAML files.
4. Replaces the root POM with the generated output and copies the modified
   source tree into the image build context.

Successful patch application and pombump execution are configuration checks,
not proof of dependency compatibility or vulnerability remediation. Review the
generated XML and pombump log, then validate the Spark build and affected runtime
behavior. Use the required OSPC SQAAS gates for security verification.

For the configuration-testing procedure, see the
[spark-pombump-testing skill](../../.github/skills/spark-pombump-testing/SKILL.md).
pombump re-serializes the POM, so textual diffs can include substantial formatting
and element-order changes in addition to the requested version changes.

Only the files directly in this directory are used by the current action;
files under `other/` are not selected.