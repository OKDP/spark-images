#
# Copyright 2026 The OKDP Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
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
ARG BASE_IMAGE=${REGISTRY}/${REPO}/spark-base:spark-${SPARK_VERSION}-scala-${SCALA_VERSION}-java-${JAVA_VERSION}

FROM $BASE_IMAGE

ARG AUTH_FILTER_VERSION=1.4.2
ARG JMX_JAVA_AGENT_VERSION=1.5.0

ENV JMX_CONF_DIR=/etc/metrics/conf/

# OKDP addons
RUN set -eux; \
    cd ${SPARK_HOME}/jars/; \
    curl -fsSL "https://repo1.maven.org/maven2/io/okdp/okdp-spark-auth-filter/${AUTH_FILTER_VERSION}/okdp-spark-auth-filter-${AUTH_FILTER_VERSION}.jar" \
      -o "okdp-spark-auth-filter-${AUTH_FILTER_VERSION}.jar"; \
    curl -fsSL "https://github.com/prometheus/jmx_exporter/releases/download/${JMX_JAVA_AGENT_VERSION}/jmx_prometheus_javaagent-${JMX_JAVA_AGENT_VERSION}.jar" \
      -o "jmx_prometheus_javaagent-${JMX_JAVA_AGENT_VERSION}.jar";

# Jmx prometheus metrics
COPY --chown=spark:spark metrics.properties ${JMX_CONF_DIR}/metrics.properties
COPY --chown=spark:spark prometheus.yaml    ${JMX_CONF_DIR}/prometheus.yaml

# Add symkink for jmx_prometheus_agent
RUN ln -s "$SPARK_HOME/jars/jmx_prometheus_javaagent-"*.jar \
          "$SPARK_HOME/jars/jmx_prometheus_javaagent.jar"

USER spark
