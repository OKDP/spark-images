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

FROM maven:3.9.13-eclipse-temurin-25 AS builder

ARG SPARK_PACKAGES="org.apache.iceberg:iceberg-spark-runtime-3.5_2.13:1.10.1,org.apache.iceberg:iceberg-aws-bundle:1.10.1,io.okdp:okdp-spark-auth-filter:1.4.2,https://github.com/prometheus/jmx_exporter/releases/download/1.5.0/jmx_prometheus_javaagent-1.5.0.jar"

WORKDIR /build

RUN bash -eu -o pipefail <<'EOF'
mkdir -p /build/target/jars

# Split SPARK_PACKAGES on commas into one item per line.
printf '%s' "${SPARK_PACKAGES:-}" | tr ',' '\n' > /build/packages.txt

# Start with an empty dependency fragment.
: > /build/dependencies.xml

# Parse entries:
# - http(s)://...                -> download directly
# - group:artifact:version       -> Maven dependency
# - group:artifact:version:type  -> Maven dependency with explicit type
# - group:artifact:version:type:classifier -> Maven dependency with type + classifier
while IFS= read -r raw; do
  item="$(printf '%s' "$raw" | sed 's/^[[:space:]]*//; s/[[:space:]]*$//')"
  [ -z "$item" ] && continue

  case "$item" in
    http://*|https://*)
      file_name="$(basename "${item%%\?*}")"
      curl -fsSL "$item" -o "/build/target/jars/${file_name}"
      ;;
    *)
      IFS=':' read -r -a parts <<< "$item"
      count="${#parts[@]}"

      if [ "$count" -lt 3 ] || [ "$count" -gt 5 ]; then
        echo "Unsupported package entry: $item" >&2
        echo "Expected Maven coordinate forms:" >&2
        echo "  groupId:artifactId:version" >&2
        echo "  groupId:artifactId:version:type" >&2
        echo "  groupId:artifactId:version:type:classifier" >&2
        echo "Or a direct http(s) jar URL." >&2
        exit 1
      fi

      group_id="${parts[0]}"
      artifact_id="${parts[1]}"
      version="${parts[2]}"
      type=""
      classifier=""

      if [ "$count" -ge 4 ]; then
        type="${parts[3]}"
      fi

      if [ "$count" -eq 5 ]; then
        classifier="${parts[4]}"
      fi

      {
        echo "    <dependency>"
        echo "      <groupId>${group_id}</groupId>"
        echo "      <artifactId>${artifact_id}</artifactId>"
        echo "      <version>${version}</version>"
        if [ -n "$type" ]; then
          echo "      <type>${type}</type>"
        fi
        if [ -n "$classifier" ]; then
          echo "      <classifier>${classifier}</classifier>"
        fi
        echo "    </dependency>"
      } >> /build/dependencies.xml
      ;;
  esac
done < /build/packages.txt

cat > /build/pom.xml <<'POM'
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>local.spark.image</groupId>
  <artifactId>dynamic-spark-runtime-dependencies</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>

  <dependencies>
POM

cat /build/dependencies.xml >> /build/pom.xml

cat >> /build/pom.xml <<'POM'
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <version>3.10.0</version>
        <executions>
          <execution>
            <id>copy-runtime-dependencies</id>
            <phase>package</phase>
            <goals>
              <goal>copy-dependencies</goal>
            </goals>
            <configuration>
              <includeScope>runtime</includeScope>
              <outputDirectory>${project.build.directory}/jars</outputDirectory>
              <excludeTransitive>false</excludeTransitive>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
POM

# Always run Maven; it succeeds even if there are no Maven dependencies.
mvn -B -q -f /build/pom.xml package
EOF

FROM $BASE_IMAGE

ENV JMX_CONF_DIR=/etc/metrics/conf/

# Copy resolved and downloaded jars into Spark.
COPY --from=builder /build/target/jars/ ${SPARK_HOME}/jars/

# JMX Prometheus metrics config
COPY --chown=spark:spark metrics.properties ${JMX_CONF_DIR}/metrics.properties
COPY --chown=spark:spark prometheus.yaml    ${JMX_CONF_DIR}/prometheus.yaml

# Add symlink for jmx_prometheus_javaagent only when present
RUN set -eux; \
    agent_file="$(find "${SPARK_HOME}/jars" -maxdepth 1 -type f -name 'jmx_prometheus_javaagent-*.jar' | head -n 1 || true)"; \
    if [ -n "${agent_file}" ]; then \
      ln -sf "${agent_file}" "${SPARK_HOME}/jars/jmx_prometheus_javaagent.jar"; \
    fi

USER spark
