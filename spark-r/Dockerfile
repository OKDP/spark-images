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
ARG BASE_IMAGE=${REGISTRY}/${REPO}/spark:spark-${SPARK_VERSION}-scala-${SCALA_VERSION}-java-${JAVA_VERSION}

FROM $BASE_IMAGE

USER root

RUN set -ex; \
    apt update; \
    apt-get install -y --no-install-recommends r-base r-base-dev; \
    rm -rf /var/lib/apt/lists/*

# debug    
RUN which R                                   # R executable exists
RUN R --version                               # R version
RUN ls -la /opt/spark/jars/ | grep -i sparkr  # SparkR JAR files
RUN ls -la /opt/spark/R/                      # SparkR library directory
# end debug

ENV R_HOME /usr/lib/R

USER spark
