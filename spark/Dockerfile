FROM eclipse-temurin:${JAVA_VERSION}-jre-jammy AS okdp_addons

ARG SPARK_VERSION=3.2.1
ARG SCALA_VERSION=2.12

RUN set -ex; \
    apt-get update; \
    apt install -y --no-install-recommends maven dos2unix

WORKDIR /workspace

COPY okdp-addons.pom .
COPY minio.pom .

# Copy patched Spark files if they exist, create empty directory otherwise
RUN mkdir -p ./spark-source
COPY patched-spark-files/ ./spark-source/ 2>/dev/null || true

# Use patched Spark parent POM if available, otherwise download from Maven Central
RUN if [ -f "./spark-source/pom.xml" ]; then \
        echo "Found patched Spark parent POM, using it"; \
        cp ./spark-source/pom.xml ./spark-parent_${SCALA_VERSION}-${SPARK_VERSION}.pom; \
    else \
        echo "No patched files found, downloading Spark parent POM from Maven Central"; \
        mvn -ntp dependency:get -DgroupId=org.apache.spark -DartifactId=spark-parent_${SCALA_VERSION} -Dversion=${SPARK_VERSION} -Dpackaging=pom; \
        mvn -ntp dependency:copy -Dartifact=org.apache.spark:spark-parent_${SCALA_VERSION}:${SPARK_VERSION}:pom -Dproject.basedir=./ -DoutputDirectory=./; \
    fi; \
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

COPY --from=okdp_addons --chown=spark:spark /workspace/target/dependency/* $SPARK_HOME/jars
COPY --chown=spark:spark metrics.properties ${JMX_CONF_DIR}/metrics.properties
COPY --chown=spark:spark prometheus.yaml    ${JMX_CONF_DIR}/prometheus.yaml

USER spark