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
ARG JAVA_VERSION=11
ARG BASE_IMAGE=eclipse-temurin:${JAVA_VERSION}-jdk-jammy 
FROM $BASE_IMAGE AS spark_builder

ARG spark_uid=185
ARG SPARK_VERSION=3.2.1
ARG HADOOP_VERSION=3.2
ARG SCALA_VERSION=2.12
ARG SPARK_REPO_URL=https://github.com/apache/spark.git

ENV SPARK_VERSION    ${SPARK_VERSION}
ENV HADOOP_VERSION   ${HADOOP_VERSION}
ENV SCALA_VERSION    ${SCALA_VERSION}
ENV SPARK_REPO_URL   ${SPARK_REPO_URL}

# Install build dependencies
RUN set -ex; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        git \
        maven \
        python3 \
        python3-pip \
        r-base \
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
        libnss-wrapper; \
    rm -rf /var/cache/apt/* && rm -rf /var/lib/apt/lists/*

# Download and build Spark from source
RUN set -ex; \
    export WORK_DIR="$(mktemp -d)"; \
    cd ${WORK_DIR}; \
    \
    echo "=== BUILDING SPARK ${SPARK_VERSION} FROM SOURCE ==="; \
    echo "Repository: ${SPARK_REPO_URL}"; \
    \
    # Determine branch/tag based on version
    MAJOR_VERSION=$(echo ${SPARK_VERSION} | cut -d. -f1); \
    MINOR_VERSION=$(echo ${SPARK_VERSION} | cut -d. -f2); \
    PATCH_VERSION=$(echo ${SPARK_VERSION} | cut -d. -f3); \
    \
    # Use exact tag if patch version exists, otherwise use branch
    if [ -n "${PATCH_VERSION}" ] && [ "${PATCH_VERSION}" != "0" ]; then \
        BRANCH="v${SPARK_VERSION}"; \
        echo "Using tag: ${BRANCH}"; \
    else \
        BRANCH="branch-${MAJOR_VERSION}.${MINOR_VERSION}"; \
        echo "Using branch: ${BRANCH}"; \
    fi; \
    \
    echo "=== CLONING SPARK REPOSITORY ==="; \
    # Shallow clone of specific branch/tag for efficiency
    git clone --depth 1 --branch ${BRANCH} ${SPARK_REPO_URL} spark-source; \
    cd spark-source; \
    \
    echo "=== REPOSITORY INFO ==="; \
    echo "Repository URL: ${SPARK_REPO_URL}"; \
    echo "Current commit: $(git rev-parse HEAD)"; \
    echo "Branch/Tag: $(git describe --tags --always)"; \
    echo "Remote origin: $(git remote get-url origin)"; \
    \
    # Determine Hadoop profile and version
    if [ "${HADOOP_VERSION}" = "3.2" ]; then \
        HADOOP_PROFILE="hadoop-3.2"; \
        HADOOP_VERSION_FULL="3.2.4"; \
    elif [ "${HADOOP_VERSION}" = "3.3" ]; then \
        HADOOP_PROFILE="hadoop-3.2"; \
        HADOOP_VERSION_FULL="3.3.6"; \
    else \
        HADOOP_PROFILE="hadoop-3.2"; \
        HADOOP_VERSION_FULL="${HADOOP_VERSION}.0"; \
    fi; \
    \
    echo "=== BUILDING WITH HADOOP ${HADOOP_VERSION_FULL} AND SCALA ${SCALA_VERSION} ==="; \
    \
    # Configure Maven memory
    export MAVEN_OPTS="-Xmx3g -XX:ReservedCodeCacheSize=1g"; \
    \
    # Build profiles - adjust based on Spark version
    BUILD_PROFILES="-Pyarn -P${HADOOP_PROFILE} -Phadoop-cloud -Phive -Phive-thriftserver"; \
    \
    # Add Kubernetes profile for versions that support it (3.0+)
    if [ "${MAJOR_VERSION}" -ge "3" ]; then \
        BUILD_PROFILES="${BUILD_PROFILES} -Pkubernetes"; \
    fi; \
    \
    # Add Scala 2.13 profile if needed
    if [ "${SCALA_VERSION}" = "2.13" ]; then \
        BUILD_PROFILES="${BUILD_PROFILES} -Pscala-2.13 -Dscala-2.13"; \
    fi; \
    \
    echo "Build profiles: ${BUILD_PROFILES}"; \
    \
    # Build Spark
    ./build/mvn ${BUILD_PROFILES} \
        -Dhadoop.version=${HADOOP_VERSION_FULL} \
        -DskipTests clean package; \
    \
    echo "=== CREATING SPARK DISTRIBUTION ==="; \
    \
    # Create distribution
    DIST_PROFILES="-P${HADOOP_PROFILE} -Phadoop-cloud -Phive -Phive-thriftserver"; \
    if [ "${MAJOR_VERSION}" -ge "3" ]; then \
        DIST_PROFILES="${DIST_PROFILES} -Pkubernetes"; \
    fi; \
    \
    ./dev/make-distribution.sh --name custom --pip --r --tgz ${DIST_PROFILES} \
        -Dhadoop.version=${HADOOP_VERSION_FULL}; \
    \
    # Move the built distribution
    mkdir -p /tmp/spark-dist; \
    mv spark-*.tgz /tmp/spark-dist/spark-custom.tgz; \
    cd /tmp/spark-dist; \
    tar -xzf spark-custom.tgz; \
    mv spark-* spark-built; \
    \
    echo "=== SPARK BUILD COMPLETED ==="; \
    echo "Built Spark version: $(cat spark-built/RELEASE | head -1)"; \
    echo "Built from commit: $(cd ${WORK_DIR}/spark-source && git rev-parse HEAD)"; \
    echo "Repository: ${SPARK_REPO_URL}"; \
    ls -la /tmp/spark-dist/; \
    \
    # Clean up build directory
    rm -rf ${WORK_DIR}

# Final runtime image
ARG BASE_IMAGE_RUNTIME=eclipse-temurin:${JAVA_VERSION}-jre-jammy 
FROM $BASE_IMAGE_RUNTIME

ARG spark_uid=185
ARG SPARK_VERSION=3.2.1
ARG HADOOP_VERSION=3.2
ARG SCALA_VERSION=2.12
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