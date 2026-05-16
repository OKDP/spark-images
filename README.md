[![ci](https://github.com/OKDP/spark-images/actions/workflows/ci.yml/badge.svg)](https://github.com/OKDP/spark-images/actions/workflows/ci.yml)
[![release-please](https://github.com/OKDP/spark-images/actions/workflows/release-please.yml/badge.svg)](https://github.com/OKDP/spark-images/actions/workflows/release-please.yml)&ensp;&ensp;
[![Release](https://img.shields.io/github/v/release/OKDP/spark-images)](https://github.com/OKDP/spark-images/releases/latest)
[![Spark](https://img.shields.io/badge/spark-3.2%20%7C%203.3%20%7C%203.4%20%7C%203.5-orange.svg)](https://spark.apache.org/)
[![License Apache2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>

<p align="center">
    <img width="400px" height="auto" src="https://okdp.io/logos/okdp-inverted.png" />
</p>

Apache Spark Docker images built from the official Spark distribution, with automatic dependency bumps and a small set of runtime jars baked in.

## What is OKDP Spark Images?

This repository builds and publishes a matrix of Apache Spark container images to `quay.io/okdp`.

- **Built from the official Spark distribution** ([source](spark-base/Dockerfile)) — Java 11/17, Scala 2.12/2.13, Hadoop 3.3.6.
- **Automatic dependency bumps** via pombump for the versions listed in [`.build/pre-build-patch-pombump.yml`](.build/pre-build-patch-pombump.yml) — the properties bumped (Log4j, Jackson, Netty, Guava, ...) are listed per Spark line in [`spark-base/spark-X.Y/pombump-properties.yaml`](spark-base/). See [Patching and Dependency Management System](#patching-and-dependency-management-system).
- **Extra runtime jars** added to the `spark` image are declared in [`.build/ci-versions.yml`](.build/ci-versions.yml) — currently includes Iceberg runtime + Iceberg AWS bundle, [`okdp-spark-auth-filter`](https://github.com/OKDP/okdp-spark-auth-filter), and the Prometheus JMX javaagent.
- **4 image variants** (`spark-base`, `spark`, `spark-py`, `spark-r`) — see the architecture diagram below.

Currently, the images are built from the [Apache Spark project distribution](https://archive.apache.org/dist/spark) and the requirement may evolve to produce them from the [source code](https://github.com/apache/spark).

<p align="center">
 <img src="docs/images/spark-images.drawio.svg">
</p>

## Image Variants

| Image          | Description                                                                                                                                                                                                                                                                       |
|:---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `JRE`          | The JRE LTS base image supported by Apache Spark depending on the version. This includes Java 11/17/21. Please, check the [reference versions](.build/reference-versions.yml) or [Apache Spark website](https://spark.apache.org/docs/latest/) for more information. |
| `spark-base`   | The Apache Spark base image with official spark binaries (scala/java) and without OKDP extensions.                                                                                                                                                                                |
| `spark`        | The Apache Spark image with official spark binaries (scala/java) and OKDP extensions.                                                                                                                                                                                             |
| `spark-py`     | The Apache Spark image with official spark binaries (scala/java), OKDP extensions and python support.                                                                                                                                                                             |
| `spark-r`      | The Apache Spark image with official spark binaries (scala/java), OKDP extensions and R support.                                                                                                                                                                                  |

## Prerequisites

- [Docker](https://www.docker.com/) (multi-stage build support — BuildKit recommended).
- Enough free disk for the image (the published `spark` image is ~3.5 GB).

## Quick Start

Pull the latest OKDP Spark image:

```sh
docker pull quay.io/okdp/spark:spark-3.5.6-scala-2.13-java-17
```

Run a `SparkPi` job in local mode to verify everything works:

```sh
docker run --rm quay.io/okdp/spark:spark-3.5.6-scala-2.13-java-17 \
  /opt/spark/bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master 'local[2]' \
  /opt/spark/examples/jars/spark-examples_2.13-3.5.6.jar 100
```

Expected last lines:
```
Pi is roughly 3.1415...
```

> Replace `spark-3.5.6-scala-2.13-java-17` with any other published tag (e.g. `spark-3.4.2-scala-2.13-java-17`). See [Tagging](#tagging) below and the [Releases](https://github.com/OKDP/spark-images/releases) page for the full matrix.

## Verify your image

The Quick Start command above is the canonical smoke test: it runs `SparkPi` in local mode using only the image (no cluster, no extra configuration). A successful run produces a `Pi is roughly 3.14...` line and the container exits with code 0.

For Kubernetes integration tests (used in CI), see the [`spark-tests-run`](.github/actions/spark-tests-run/) action.

# Tagging

The project builds the images with a long format tags. Each tag combines multiple compatible versions combinations.

There are multiple tags levels and the format to use depends on your convenience in term of stability and reproducibility.

The images are pushed to [quay.io/okdp](https://quay.io/organization/okdp) repository with the following [tags](.build/images.yml):

| Images              | Tags                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|:--------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| spark-base, spark | spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION></br></br>spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE></br></br>spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION></br></br>spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION>                                                                                                     |
| spark-py          | spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION></br></br>spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE></br></br>spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION></br></br>spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION> |
| spark-r           | spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION></br></br> spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE></br></br>spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION></br></br>spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION>                                        |

> [!NOTE]
> 1. `<RELEASE_VERSION>` corresponds to the Github [release version](https://github.com/OKDP/spark-images/releases) or [git tag](https://github.com/OKDP/spark-images/tags) without the leading `v`.
>    Ex.: 2.1.0
>
> 2. `<BUILD_DATE>` corresponds to the images build date with the `YYYY-MM-DD` format. The latest release tag is rebuilt every week to ensure the OS image is up to date against the latest security updates.
>
>    You may need to switch to the latest release version if your are using the long form tag image with a `<RELEASE_VERSION>`. Please, check the [changelog](https://github.com/OKDP/spark-images/releases) to see the notable impacts.
>
>    An example of `spark-py` image with a long form tag including `spark/java/scala/python` compatible versions and a `<BUILD_DATE>` with a `<RELEASE_VERSION>` is:
>
>    `quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.13-java-17-2026-05-12-2.1.0`.
>
>    The corresponding changelog is [releases/tag/v2.1.0](https://github.com/OKDP/spark-images/releases/tag/v2.1.0).
>
> 3. You can also use the latest tag without `<BUILD_DATE>` and `<RELEASE_VERSION>` which is always up to date with the latest security updates.
>
>    An example of `spark-py` image with the latest tag is: `quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.13-java-17`
>

# Patching and Dependency Management System

This project automatically applies security fixes and dependency updates to Spark source code during builds using a patch and pombump system.

**Key Features:**
- ✅ **Source code patches** for critical security fixes
- ✅ **Automated dependency updates** via pombump
- ✅ **Version-specific configurations** 
- ✅ **Build optimization** and compatibility

## How It Works

### Configuration-Based Processing

The system uses `.build/pre-build-patch-pombump.yml` to determine which Spark versions should receive patches and/or dependency updates:

```yaml
controls:
  - spark_version: "3.4.1"
    python_version: "3.11"
    java_version: "17"
    hadoop_version: "3.3.6"
    patch_files: []  # No source patches needed, but pombump will run
```

### Processing Logic

**If a Spark version is present in the configuration file:**

1. **Source Download**: The system downloads the Spark source code
2. **Patch Application**: Applies any source code patches (if `patch_files` is not empty)
3. **Dependency Updates**: Runs pombump to update Maven dependencies to secure versions
4. **Build Context**: Uses the patched/updated source for Docker builds

**If a Spark version is not in the configuration:**
- Uses original Spark distribution without modifications

### Pombump Dependency Management

For versions in the configuration, pombump automatically updates dependencies to secure versions:

```yaml
# From pombump-properties.yaml
- property: log4j.version
  value: "2.25.0"  # Updates to secure Log4j version
- property: fasterxml.jackson.version  
  value: "2.14.2"  # Updates Jackson for security
```

This ensures all builds use the latest secure dependency versions, even without source code changes.

📖 **[Read the full patching documentation →](PATCH-POMBUMP.md)**

**Quick Reference:**
- Patch configuration: [`.build/pre-build-patch-pombump.yml`](.build/pre-build-patch-pombump.yml)
- Patch files: [`spark-base/spark-X.Y/`](spark-base/)
- Application logic: [`.github/actions/patch-pombump/`](.github/actions/patch-pombump/)

# Alternatives

- [Official Apache Spark images](https://github.com/apache/spark-docker)

---

**Built 🚀 for the OKDP Community**
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>
