[![ci](https://github.com/okdp/spark-images/actions/workflows/ci.yml/badge.svg)](https://github.com/okdp/spark-images/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/okdp/spark-images)](https://github.com/okdp/spark-images/releases/latest)
[![License Apache2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)


Collection of [Apache Spark](https://spark.apache.org/) docker images for [OKDP Platform](https://okdp.io/).

Currently, the images are built from the [Apache Spark project distribution](https://archive.apache.org/dist/spark) and the requirement may evolve to produce them from the [source code](https://github.com/apache/spark).

The image relashionship is described by the following diagram:

<p align="center">
 <img src="docs/images/spark-images.drawio.svg">
</p>




| Image          | Description                                                                                                                                                                                                                                                                       |
|:---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `JRE`          | The JRE LTS base image supported by Apache Spark depending on the version. This includes Java 11/17/21. Please, check the [reference versions](.build/reference-versions.yml) or [Apache Spark website](https://spark.apache.org/docs/latest/) for more information. |
| `spark-base`   | The Apache Spark base image with official spark binaries (scala/java) and without OKDP extensions.                                                                                                                                                                                |
| `spark`        | The Apache Spark image with official spark binaries (scala/java) and OKDP extensions.                                                                                                                                                                                             | 
| `spark-py`     | The Apache Spark image with official spark binaries (scala/java), OKDP extensions and python support.                                                                                                                                                                             | 
| `spark-r`      | The Apache Spark image with official spark binaries (scala/java), OKDP extensions and R support.                                                                                                                                                                                  | 

# Tagging

The project builds the images with a long format tags. Each tag combines multiple compatible versions combinations.

There are multiple tags levels and the format to use depends on your convenience in term of stability and reproducibility.

The images are pushed to [quay.io/okdp](https://quay.io/organization/okdp) repository with the following [tags](.build/images.yml):

| Images              | Tags                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|:--------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| spark-base, spark | spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION></br></br>spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE></br></br>spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION></br></br>spark-<SPARK_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION>                                                                                                     |
| spark-py          | spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION></br></br>spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE></br></br>spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION></br></br>spark-<SPARK_VERSION>-python-<PYTHON_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION> |
| spark-r           | spark-<SPARK_VERSION>-r-<R_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION></br></br> spark-<SPARK_VERSION>-r-<R_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE></br></br>spark-<SPARK_VERSION>-r-<R_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<RELEASE_VERSION></br></br>spark-<SPARK_VERSION>-r-<R_VERSION>-scala-<SCALA_VERSION>-java-<JAVA_VERSION>-<BUILD_DATE>-<RELEASE_VERSION>                                        |

> [!NOTE]
> 1. `<RELEASE_VERSION>` corresponds to the Github [release version](https://github.com/okdp/spark-images/releases) or [git tag](https://github.com/okdp/spark-images/tags) without the leading `v`.
>    Ex.: 1.0.0
> 
> 2. `<BUILD_DATE>` corresponds to the images build date with the `YYYY-MM-DD` format. The latest release tag is rebuilt every week to ensure the OS image is up to date against the latest security updates.
> 
>    You may need to switch to the latest release version if your are using the long form tag image with a `<RELEASE_VERSION>`. Please, check the [changelog](https://github.com/okdp/spark-images/releases) to see the notable impacts.
>
>    An example of `py-spark` image with a long form tag including `spark/java/scala/python` compatible versions and a `<BUILD_DATE>` with a `<RELEASE_VERSION>` is: 
> 
>    `quay.io/okdp/spark-py:spark-3.5.1-python-3.11-scala-2.13-java-17-2024-04-04-1.0.0`.
>
>    The corresponding changelog is [releases/tag/v1.0.0](https://github.com/okdp/spark-images/releases/tag/v1.0.0).
>
> 3. You can also use the latest tag without `<BUILD_DATE>` and `<RELEASE_VERSION>` which is always up to date with the latest security updates. 
> 
>    An example of `py-spark` image with the latest tag is: `quay.io/okdp/spark-py:spark-3.5.1-python-3.11-scala-2.13-java-17`
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
  - spark_version: [3.4.1]
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

- [Official images](https://github.com/apache/spark-docker)

