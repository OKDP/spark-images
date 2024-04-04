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

# Alternatives

- [Official images](https://github.com/apache/spark-docker)

