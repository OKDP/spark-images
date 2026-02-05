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
ARG BUILDPLATFORM=linux/amd64
ARG JAVA_VERSION=17
FROM --platform=${BUILDPLATFORM} eclipse-temurin:${JAVA_VERSION}-jdk-jammy AS spark_builder

ARG spark_uid=185
ARG SPARK_VERSION=3.5.1
ARG HADOOP_VERSION=3.3.6
ARG SCALA_VERSION=2.13
ARG SPARK_REPO_URL=https://github.com/apache/spark.git

ENV SPARK_VERSION    ${SPARK_VERSION}
ENV HADOOP_VERSION   ${HADOOP_VERSION}
ENV SCALA_VERSION    ${SCALA_VERSION}
ENV SPARK_REPO_URL   ${SPARK_REPO_URL}

# ENV MAVEN_OPTS="-Xss64m -Xmx2g -XX:ReservedCodeCacheSize=1g"
# ENV MAVEN_OPTS="-Xss64m -Xmx3g -XX:ReservedCodeCacheSize=1g"
ENV MAVEN_OPTS="-Xss64m -Xmx4g -XX:ReservedCodeCacheSize=1g -XX:+UseG1GC -XX:G1HeapRegionSize=16m"
ARG SPARK_BUILD_ARGS="\
    --name custom \
    --pip \
    -Pkubernetes \
    -Phadoop-cloud \
    -Pscala-${SCALA_VERSION} \
    -Dhadoop.version=${HADOOP_VERSION} \
    -Phive \
    -Phive-thriftserver \
    -DskipTests \
    -Dmaven.source.skip=true \
    -Dmaven.site.skip=true \
    -Dmaven.javadoc.skip=true \
    -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn -B \
    -Psparkr"

# Install build dependencies
RUN set -ex; \
    apt-get update; \
    export DEBIAN_FRONTEND=noninteractive; \
    apt-get install -y --no-install-recommends \
        git \
        maven \
        python3 \
        python3-pip \
        build-essential \
        curl \
        bash \
        tini \
        libc6 \
        libpam-modules \
        krb5-user \
        libnss3 \
        procps \
        net-tools \
        gosu \
        libnss-wrapper \
        r-base \
        r-base-dev; \
    rm -rf /var/cache/apt/* && rm -rf /var/lib/apt/lists/*

ENV WORK_DIR=/tmp/spark-build
ENV SPARK_SOURCE=/tmp/spark-build/spark-source

WORKDIR ${WORK_DIR}

# Copy everything from build context, including patches if they exist
COPY . ${WORK_DIR}/

# Download and build Spark from source
RUN set -ex; \
    export WORK_DIR="$(mktemp -d)"; \
    cd ${WORK_DIR}; \
    echo "=== BUILDING SPARK ${SPARK_VERSION} FROM SOURCE ==="; \
    # Check if patched files exist and are not empty
    if [ -d "${WORK_DIR}/patched-spark-files" ] && [ "$(ls -A ${WORK_DIR}/patched-spark-files 2>/dev/null)" ]; then \
        echo "=== USING PATCHED SPARK SOURCE FILES ==="; \
        echo "Found patched source files, using them instead of cloning"; \
        cp -r ${WORK_DIR}/patched-spark-files ${SPARK_SOURCE}; \
        cd ${SPARK_SOURCE}; \
        chmod +x build/mvn dev/make-distribution.sh; \
        echo "Patched Spark source ready for build"; \
    else \
        echo "=== CLONING SPARK REPOSITORY ==="; \
        echo "Repository: ${SPARK_REPO_URL}"; \
        echo "Version: ${SPARK_VERSION}"; \
        git clone --depth 1 --branch v${SPARK_VERSION} ${SPARK_REPO_URL} ${SPARK_SOURCE}; \
        cd ${SPARK_SOURCE}; \
        echo "Spark v${SPARK_VERSION} cloned successfully"; \
    fi

RUN set -ex; \
    echo "=== CONFIGURING SCALA VERSION ==="; \
    echo "Target Scala version: ${SCALA_VERSION}"; \
    if [ "${SCALA_VERSION}" = "2.13" ]; then \
        echo "Switching to Scala 2.13 using official script"; \
        cd ${SPARK_SOURCE}; \
        ./dev/change-scala-version.sh 2.13; \
        echo "✅ Scala version changed to 2.13"; \
    fi

RUN set -ex; \
    echo "=== BUILDING SPARK DISTRIBUTION ==="; \
    cd ${SPARK_SOURCE}; \
    ./dev/make-distribution.sh ${SPARK_BUILD_ARGS}; \
    mkdir -p /tmp/spark-dist; \
    mv dist /tmp/spark-dist/spark-built; \
    \
    echo "=== SPARK BUILD COMPLETED ==="; \
    echo "Built Spark version: $(cat /tmp/spark-dist/spark-built/RELEASE | head -1)"; \
    ls -la /tmp/spark-dist/

# Final runtime image
ARG JAVA_VERSION=17
FROM eclipse-temurin:${JAVA_VERSION}-jre-jammy

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

ARG spark_uid=185
ARG SPARK_VERSION=3.5.1
ARG HADOOP_VERSION=3.3.6
ARG SCALA_VERSION=2.13
ARG SPARK_REPO_URL=https://github.com/apache/spark.git

ENV SPARK_HOME       /opt/spark
ENV SPARK_CONF_DIR   ${SPARK_HOME}/conf
ENV SPARK_VERSION    ${SPARK_VERSION}
ENV HADOOP_VERSION   ${HADOOP_VERSION}
ENV SCALA_VERSION    ${SCALA_VERSION}
ENV SPARK_REPO_URL   ${SPARK_REPO_URL}

# Create spark user and install runtime dependencies
RUN groupadd --system --gid=${spark_uid} spark && \
    useradd --system --uid=${spark_uid} --gid=spark spark

RUN set -ex; \
    apt-get update; \
    ln -s /lib /lib64; \
    export DEBIAN_FRONTEND=noninteractive; \
    apt install -y --no-install-recommends \
        bash \
        tini \
        libc6 \
        libpam-modules \
        krb5-user \
        libnss3 \
        procps \
        net-tools \
        gosu \
        libnss-wrapper \
        curl \
        python3 \
        python3-pip; \
    mkdir -p ${SPARK_HOME}; \
    mkdir -p ${SPARK_HOME}/work-dir; \
    chmod g+w ${SPARK_HOME}/work-dir; \
    chown -R spark:spark ${SPARK_HOME}; \
    rm /bin/sh; \
    ln -sv /bin/bash /bin/sh; \
    echo "auth required pam_wheel.so use_uid" >> /etc/pam.d/su; \
    chgrp root /etc/passwd && chmod ug+rw /etc/passwd; \
    rm -rf /var/cache/apt/* && rm -rf /var/lib/apt/lists/*

# Copy built Spark from builder stage
COPY --from=spark_builder /tmp/spark-dist/spark-built/ ${SPARK_HOME}/

# Setup Spark directory permissions and cleanup
RUN set -ex; \
    chown -R spark:spark ${SPARK_HOME}/; \
    # Move decom.sh if it exists (Kubernetes deployments)
    if [ -f "${SPARK_HOME}/kubernetes/dockerfiles/spark/decom.sh" ]; then \
        mv ${SPARK_HOME}/kubernetes/dockerfiles/spark/decom.sh /opt/; \
        chmod a+x /opt/decom.sh; \
    fi; \
    # Move tests if they exist
    if [ -d "${SPARK_HOME}/kubernetes/tests" ]; then \
        mv ${SPARK_HOME}/kubernetes/tests ${SPARK_HOME}/; \
    fi; \
    # Cleanup unnecessary directories
    rm -rf ${SPARK_HOME}/conf 2>/dev/null || true; \
    rm -rf ${SPARK_HOME}/yarn 2>/dev/null || true; \
    rm -rf ${SPARK_HOME}/kubernetes 2>/dev/null || true; \
    # Verify Spark installation
    echo "=== SPARK INSTALLATION VERIFIED ==="; \
    echo "Spark version: $(cat ${SPARK_HOME}/RELEASE | head -1)"; \
    echo "Built from repository: ${SPARK_REPO_URL}"; \
    echo "Spark home contents:"; \
    ls -la ${SPARK_HOME}/

COPY entrypoint.sh /opt/entrypoint.sh
RUN chmod a+x /opt/entrypoint.sh

WORKDIR ${SPARK_HOME}/work-dir

USER spark

ENTRYPOINT [ "/opt/entrypoint.sh" ]
