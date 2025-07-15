#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
ARG SPARK_VERSION=3.2.1
ARG HADOOP_VERSION=3.2
ARG SCALA_VERSION=2.12
ARG JAVA_VERSION=11

ARG REGISTRY=quay.io
ARG REPO=okdp
ARG BASE_IMAGE=${REGISTRY}/${REPO}/spark:base-spark-${SPARK_VERSION}-scala-${SCALA_VERSION}-java-${JAVA_VERSION}

FROM eclipse-temurin:${JAVA_VERSION}-jre-jammy AS okdp_addons
ARG SPARK_VERSION=3.2.1
ARG SCALA_VERSION=2.12

RUN set -ex; \
    apt-get update; \
    apt install  -y --no-install-recommends maven dos2unix

WORKDIR /workspace

COPY okdp-addons.pom .
COPY minio.pom .

# Copy the entire version-specific folder (if it exists)
COPY ${SPARK_VERSION}/ /tmp/

# Conditionally apply patch if all files and versions match
RUN if [ -f /tmp/patch.diff ] && [ -f /tmp/pom-bump.properties ]; then \
      . /tmp/pom-bump.properties; \
      if [ "$spark_version" = "$SPARK_VERSION" ] && \
         [ "$hadoop_version" = "$HADOOP_VERSION" ] && \
         [ "$scala_version" = "$SCALA_VERSION" ] && \
         [ "$java_version" = "$JAVA_VERSION" ]; then \
        echo "All versions match, applying patch"; \
        patch -p1 < /tmp/patch.diff; \
      else \
        echo "Version mismatch, not applying patch"; \
      fi; \
    else \
      echo "No patch or pom-bump.properties found, skipping patch step"; \
    fi

# The setup consumes less space compare to inheriting from the parent pom
# Handles the transitive dependencies versions through the pom
# Manage Java AWS SDK v1 (hadoop <3.4)/V2 (hadoop >=3.4)
# Some pom.xml versions comes with Control M
# Minio and AWS profiles are mutually exclusive: aws includes minio
RUN mvn -ntp dependency:get -DgroupId=org.apache.spark -DartifactId=spark-parent_${SCALA_VERSION} -Dversion=${SPARK_VERSION} -Dpackaging=pom; \
    mvn -ntp dependency:copy -Dartifact=org.apache.spark:spark-parent_${SCALA_VERSION}:${SPARK_VERSION}:pom -Dproject.basedir=./ -DoutputDirectory=./; \
    dos2unix spark-parent_${SCALA_VERSION}-${SPARK_VERSION}.pom; \
    HADOOP_VERSION=$(grep "<hadoop.version>" spark-parent_${SCALA_VERSION}-${SPARK_VERSION}.pom | tr -d ' ' | sed -e 's/^ *<hadoop.version>\(.*\)<\/hadoop.version> *$/\1/' | sort -rn | head -n 1); \
    mvn -ntp dependency:get -DgroupId=org.apache.hadoop -DartifactId=hadoop-aws -Dversion=${HADOOP_VERSION} -Dpackaging=pom; \
    mvn -ntp dependency:copy -Dartifact=org.apache.hadoop:hadoop-aws:${HADOOP_VERSION}:pom -Dproject.basedir=./ -DoutputDirectory=./; \
    AWS_JAVA_SDK_VERSION=$(mvn -ntp dependency:tree -f hadoop-aws-${HADOOP_VERSION}.pom | grep -E "(com.amazonaws|software.amazon.awssdk):(aws-java-sdk-bundle|bundle):jar:.*:compile" | awk '{ print $NF }' | awk -F: '{ print $4 }'); \
    mvn -ntp clean install \
        -Daws-java-sdk.version=${AWS_JAVA_SDK_VERSION} \
        -Daws-sdk-profile.version=v$(echo ${AWS_JAVA_SDK_VERSION} | cut -d '.' -f 1) \
        -f minio.pom; \
    mvn -ntp clean dependency:copy-dependencies \
        -Dspark.version=${SPARK_VERSION} \
        -Dscala.version=${SCALA_VERSION} \
        -Dhadoop.version=${HADOOP_VERSION} \
        -Daws-java-sdk.version=${AWS_JAVA_SDK_VERSION} \
        -Pminio \
        -f okdp-addons.pom

FROM $BASE_IMAGE

ENV JMX_CONF_DIR     /etc/metrics/conf/

# OKDP addons
COPY --from=okdp_addons --chown=spark:spark /workspace/target/dependency/* $SPARK_HOME/jars

# Jmx prometheus metrics
COPY --chown=spark:spark metrics.properties ${JMX_CONF_DIR}/metrics.properties
COPY --chown=spark:spark prometheus.yaml    ${JMX_CONF_DIR}/prometheus.yaml

USER spark

