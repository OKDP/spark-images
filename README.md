<!-- Section 0 — Product image -->

<img src="https://spark.apache.org/images/spark-logo-trademark.png" alt="Apache Spark" height="48" align="right" />

<!-- Section 1 — Badges -->

[![ci](https://github.com/OKDP/spark-images/actions/workflows/ci.yml/badge.svg)](https://github.com/OKDP/spark-images/actions/workflows/ci.yml)
[![release-please](https://github.com/OKDP/spark-images/actions/workflows/release-please.yml/badge.svg)](https://github.com/OKDP/spark-images/actions/workflows/release-please.yml)
[![Release](https://img.shields.io/github/v/release/OKDP/spark-images)](https://github.com/OKDP/spark-images/releases/latest)
[![Spark](https://img.shields.io/badge/spark-3.2%20%7C%203.3%20%7C%203.4%20%7C%203.5-orange.svg)](https://spark.apache.org/)
[![License Apache2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>

<!-- Section 2 — Project name + short description -->

# OKDP Spark Images

Apache Spark Docker images built from the official Spark distribution, with automatic dependency bumps and a small set of runtime jars baked in. Published to [`quay.io/okdp`](https://quay.io/organization/okdp).

<!-- Section 3 — What the project does -->

## What it does

- **Builds the official Apache Spark distribution from source** ([`spark-base/Dockerfile`](spark-base/Dockerfile)) for Java 11/17, Scala 2.12/2.13 and Hadoop 3.3.6 — see the supported combinations in [`.build/reference-versions.yml`](.build/reference-versions.yml).
- **Applies security patches and dependency bumps** during the build (Log4j, Jackson, Netty, Guava, AWS SDK, Jetty, …) via the `pombump` system declared in [`.build/pre-build-patch-pombump.yml`](.build/pre-build-patch-pombump.yml); the full property list is in [`spark-base/spark-X.Y/pombump-properties.yaml`](spark-base/). See [PATCH-POMBUMP.md](PATCH-POMBUMP.md) for the full mechanism.
- **Bakes a set of extra runtime jars** into the `spark` image (Iceberg runtime + Iceberg AWS bundle, [`okdp-spark-auth-filter`](https://github.com/OKDP/okdp-spark-auth-filter), Prometheus JMX javaagent) — list declared in [`.build/ci-versions.yml`](.build/ci-versions.yml).
- **Ships 4 image variants** (`spark-base`, `spark`, `spark-py`, `spark-r`) — see the inheritance diagram below.

<!-- Section 4 — Architecture -->

## Architecture

<p align="center">
 <img src="docs/images/spark-images.drawio.svg" alt="Image inheritance chain">
</p>

<!-- Section 5 — Prerequisites -->

## Prerequisites

- [Docker](https://www.docker.com/) (multi-stage build support — BuildKit recommended).
- Enough free disk for the image (the published `spark` image is ~3.5 GB).

<!-- Section 6 — Quick Start -->

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

### Expected result

The container prints standard Spark INFO lines and finishes with:

```
Pi is roughly 3.14...
```

Container exit code: `0`. End-to-end run takes ~10 seconds on a recent laptop.

<!-- Section 7 — Installation -->

## Installation

The images are meant to be used in three modes; pick the one matching your deployment:

### 1. Local mode (one-shot job on a single machine)

Equivalent to the Quick Start above. Pull the `spark`, `spark-py` or `spark-r` image and invoke `spark-submit --master 'local[N]'`.

### 2. Kubernetes mode (Spark-on-Kubernetes)

The image entrypoint ([`spark-base/entrypoint.sh`](spark-base/entrypoint.sh)) implements the `driver` / `executor` commands used by `spark-submit --master k8s://…`. In a cluster:

```sh
spark-submit \
  --master k8s://https://<api-server> \
  --deploy-mode cluster \
  --conf spark.kubernetes.container.image=quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.13-java-17 \
  --class <your.MainClass> \
  local:///path/to/your.jar
```

The OKDP Sandbox runs Spark this way through the [Spark Operator](https://github.com/kubeflow/spark-operator) — see [OKDP integration](#okdp-integration).

### 3. Pass-through mode (any other command)

When the first argument is neither `driver` nor `executor`, the entrypoint exec's the command verbatim (see [`spark-base/entrypoint.sh:138-141`](spark-base/entrypoint.sh)). Useful for `spark-shell`, `pyspark`, debugging:

```sh
docker run --rm -it quay.io/okdp/spark:spark-3.5.6-scala-2.13-java-17 /opt/spark/bin/spark-shell
```

<!-- Section 8 — Configuration -->

## Configuration

### Build arguments

Passed via `docker build --build-arg NAME=value`. Defaults are the ones declared in the `Dockerfile`; the CI always overrides them via the matrix in [`.build/ci-versions.yml`](.build/ci-versions.yml).

| ARG               | Where                              | Default                                                          | Description                                                              |
|:------------------|:-----------------------------------|:-----------------------------------------------------------------|:-------------------------------------------------------------------------|
| `SPARK_VERSION`   | all `Dockerfile`s                  | `3.5.1` in [`spark-base`](spark-base/Dockerfile), `3.2.1` in `spark`/`spark-py`/`spark-r` | Apache Spark version to build / inherit from.                            |
| `HADOOP_VERSION`  | all `Dockerfile`s                  | `3.3.6` in [`spark-base`](spark-base/Dockerfile), `3.2` in `spark`/`spark-py`/`spark-r`   | Hadoop version used for the `-Phadoop-cloud` profile.                    |
| `SCALA_VERSION`   | all `Dockerfile`s                  | `2.13` in [`spark-base`](spark-base/Dockerfile), `2.12` in `spark`/`spark-py`/`spark-r`   | Scala binary version (`2.12` or `2.13`).                                 |
| `JAVA_VERSION`    | all `Dockerfile`s                  | `17` in [`spark-base`](spark-base/Dockerfile), `11` in `spark`/`spark-py`/`spark-r`       | JRE major version (`11`, `17`, …).                                       |
| `PYTHON_VERSION`  | `spark-py/Dockerfile`              | `3.11`                                                           | Python version installed via `apt-get install python${PYTHON_VERSION}`. |
| `SPARK_PACKAGES`  | `spark/Dockerfile`                 | *(Iceberg + okdp-spark-auth-filter + JMX javaagent — see [`spark/Dockerfile`](spark/Dockerfile))* | Comma-separated list of Maven coordinates or jar URLs baked into the image. |
| `BASE_IMAGE`      | `spark`, `spark-py`, `spark-r`     | derived from `SPARK_VERSION`/`SCALA_VERSION`/`JAVA_VERSION`      | Override the parent image (useful for local re-builds). |
| `SPARK_REPO_URL`  | `spark-base/Dockerfile`            | `https://github.com/apache/spark.git`                            | Git URL Spark source is cloned from when no patched files are provided. |

> [!NOTE]
> The CI always overrides these defaults via the matrix in [`.build/ci-versions.yml`](.build/ci-versions.yml), so the defaults only matter for local `docker build` invocations.

### Runtime environment variables

Read by [`spark-base/entrypoint.sh`](spark-base/entrypoint.sh) and recognised by Spark itself. Pass with `docker run -e NAME=value …`.

| ENV                          | Description                                                                                          |
|:-----------------------------|:-----------------------------------------------------------------------------------------------------|
| `SPARK_EXTRA_CLASSPATH`      | Extra entries appended to `SPARK_CLASSPATH`.                                                          |
| `HADOOP_CONF_DIR`            | Prepended to `SPARK_CLASSPATH` if set — point to your Hadoop config directory.                       |
| `HADOOP_HOME`                | When set and `SPARK_DIST_CLASSPATH` is empty, expands to `$HADOOP_HOME/bin/hadoop classpath`.        |
| `SPARK_CONF_DIR`             | Defaults to `$SPARK_HOME/conf` otherwise.                                                            |
| `SPARK_DIST_CLASSPATH`       | If unset and `HADOOP_HOME` is set, populated from `hadoop classpath`.                                |
| `PYSPARK_PYTHON`             | Path to the Python executor binary (PySpark only).                                                   |
| `PYSPARK_DRIVER_PYTHON`      | Path to the Python driver binary (PySpark only).                                                     |
| `SPARK_DRIVER_BIND_ADDRESS`  | Bind address used in `driver` mode (Spark-on-Kubernetes).                                            |
| `R_HOME`, `R_LIBS`           | Pre-set in the `spark-r` image (`/usr/lib/R`, `/opt/spark/R/lib`).                                  |
| `JMX_CONF_DIR`               | Pre-set to `/etc/metrics/conf/` — host the `prometheus.yaml` and `metrics.properties` bundled in.    |

<!-- Section 10 — Images -->

## Images

The repository builds 4 image variants, all published to [`quay.io/okdp`](https://quay.io/organization/okdp):

| Image          | Description                                                                                                                                                                                                                                                                       |
|:---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `JRE`          | The JRE LTS base image supported by Apache Spark depending on the version (Java 11/17/21). See [reference versions](.build/reference-versions.yml) or the [Apache Spark website](https://spark.apache.org/docs/latest/) for more information.                                     |
| `spark-base`   | The Apache Spark base image with official spark binaries (scala/java) and **without** OKDP extensions.                                                                                                                                                                            |
| `spark`        | The Apache Spark image with official spark binaries (scala/java) **and** OKDP extensions (Iceberg, auth filter, JMX).                                                                                                                                                              |
| `spark-py`     | Same as `spark` + Python support.                                                                                                                                                                                                                                                  |
| `spark-r`      | Same as `spark` + R support.                                                                                                                                                                                                                                                       |

### Tagging

The project builds the images with long-format tags combining multiple compatible version components. The format to use depends on your stability vs. reproducibility tradeoff.

| Image             | Tags                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|:------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `spark-base`, `spark` | `spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>`</br></br>`spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>`</br></br>`spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION>`</br></br>`spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION>`                                                                                                |
| `spark-py`        | `spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>`</br></br>`spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>`</br></br>`spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION>`</br></br>`spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION>` |
| `spark-r`         | `spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION>`</br></br>`spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>`</br></br>`spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION>`</br></br>`spark-<SPARK_VERSION>-r--scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION>`                                                                                    |

> [!NOTE]
> 1. `<RELEASE_VERSION>` corresponds to the [GitHub release version](https://github.com/OKDP/spark-images/releases) or [git tag](https://github.com/OKDP/spark-images/tags) without the leading `v` (e.g. `2.1.0`).
> 2. `<BUILD_DATE>` is the image build date (`YYYY-MM-DD`). The latest release tag is rebuilt every week to ship the latest OS / dependency security updates.
> 3. A full long-form `spark-py` tag is for example: `quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.13-java-17-2026-05-26-2.1.0` — corresponding to release [`v2.1.0`](https://github.com/OKDP/spark-images/releases/tag/v2.1.0).
> 4. The short tag without `<BUILD_DATE>` and `<RELEASE_VERSION>` always points to the latest rebuild, e.g. `quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.13-java-17`.

<!-- Section 11 — OKDP integration -->

## OKDP integration

These images are used by [`OKDP/okdp-sandbox`](https://github.com/OKDP/okdp-sandbox) — the OKDP local Kubernetes data platform — through:

- [`packages/okdp-packages/spark-operator/spark-operator.yaml`](https://github.com/OKDP/okdp-sandbox/blob/main/packages/okdp-packages/spark-operator/spark-operator.yaml) — pins `quay.io/okdp/spark-py:spark-3.5.6-python-3.11-scala-2.12-java-17` as the default image for jobs submitted via the Spark Operator.
- [`packages/okdp-packages/spark-history-server/spark-history-server.yaml`](https://github.com/OKDP/okdp-sandbox/blob/main/packages/okdp-packages/spark-history-server/spark-history-server.yaml) — uses the `quay.io/okdp/spark` image as the base for the history server pod.

<!-- Section 13 — Build -->

## Build

The whole build matrix runs on GitHub Actions and is split into:

| Workflow                                                                                       | Trigger                                            | What it does                                                                                                                                              |
|:-----------------------------------------------------------------------------------------------|:---------------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------|
| [`ci.yml`](.github/workflows/ci.yml)                                                           | PR + push on `main`                                | Builds the matrix declared in [`.build/ci-versions.yml`](.build/ci-versions.yml) into the GHCR CI registry and runs the K8s integration tests.            |
| [`publish.yml`](.github/workflows/publish.yml)                                                 | Weekly cron (Tue 05:00 UTC) + `workflow_dispatch`  | Rebuilds the latest GitHub release across the full [`.build/release-versions.yml`](.build/release-versions.yml) matrix and pushes to `quay.io/okdp/*`.    |
| [`release-please.yml`](.github/workflows/release-please.yml)                                   | push on `main`                                     | Generates release PRs and tags via [release-please](https://github.com/googleapis/release-please).                                                         |
| [`sign-images.yml`](.github/workflows/sign-images.yml)                                         | After `ci` workflow runs                           | Signs the produced images with [Cosign](https://github.com/sigstore/cosign).                                                                              |
| [`build-image-template.yml`](.github/workflows/build-image-template.yml)                       | called by `ci.yml` / `publish.yml`                 | Reusable workflow: builds, tests and pushes a single (image × spark × scala × java) combination.                                                          |
| [`build-images-template.yml`](.github/workflows/build-images-template.yml)                     | called by `ci.yml` / `publish.yml`                 | Reusable workflow: chains the 4 image variants (`spark-base` → `spark` → `spark-py`, `spark-r`) for one Spark line.                                       |
| [`build-upload-spark-dist.yml`](.github/workflows/build-upload-spark-dist.yml)                 | called by the build pipeline                       | Extracts the Spark distribution tarball from the built image and uploads it as a workflow artifact.                                                       |

Composite actions used by the workflows live under [`.github/actions/`](.github/actions/): `spark-version-matrix`, `spark-image-tag`, `setup-buildx`, `setup-kind`, `spark-tests-prepare`, `spark-tests-run`, `patch-pombump`, `free-disk-space`.

### Patching and dependency management

The build pipeline applies source patches and bumps Maven dependencies to secure versions before assembling the Spark distribution. The full mechanism (configuration, patch files, pombump-properties, processing logic) is documented in **[PATCH-POMBUMP.md](PATCH-POMBUMP.md)**.

<!-- Section 14 — Test -->

## Test

A local smoke test is documented in the [Quick Start](#quick-start) section above.

In CI, every PR runs the upstream [Apache Spark Kubernetes integration tests](https://github.com/apache/spark/tree/master/resource-managers/kubernetes/integration-tests) against the freshly built images on a [kind](https://kind.sigs.k8s.io/) cluster. The runner is the [`spark-tests-run`](.github/actions/spark-tests-run/action.yml) composite action, which:

- Loads the CI image into the kind cluster.
- Switches the Spark source tree to the target Scala version (`./dev/change-scala-version.sh`).
- Invokes `build/sbt 'kubernetes-integration-tests/testOnly -- -z "Run SparkPi"'` with one step per image variant (`spark-base` / `spark`, `spark-py`, `spark-r`).

### Expected result

All four `Run SparkPi` integration steps pass on every Spark × Scala × Java combination declared in [`.build/ci-versions.yml`](.build/ci-versions.yml). The matrix wiring lives in [`ci.yml`](.github/workflows/ci.yml).

<!-- Section 16 — Contributing + License -->

## Contributing & License

Contributions follow the [OKDP contribution guide](https://github.com/OKDP/.github/blob/main/CONTRIBUTING.md) (fork-based workflow, Conventional Commits, PR linked to an issue).

Released under the [Apache License 2.0](LICENSE).

<!-- Section 17 — Footer -->

---

**Built 🚀 for the OKDP Community**
<a href="https://okdp.io">
  <img src="https://okdp.io/logos/okdp-notext.svg" height="20px" style="margin: 0 2px;" />
</a>
