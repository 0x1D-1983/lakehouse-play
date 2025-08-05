# Use Flink 1.20, as Flink 2.0 doesn't have a compatible Iceberg version yet.  
FROM flink:1.20-java17  
  
# Add Iceberg connector + Hadoop libraries + AWS SDK  
# Unfortunately, the Iceberg Flink connector still requires Hadoop libraries
RUN cd /opt/flink/lib/ && \  
    curl -LO https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-flink-runtime-1.20/1.9.1/iceberg-flink-runtime-1.20-1.9.1.jar && \  
    curl -LO https://repo1.maven.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.8.3-10.0/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar && \  
    curl -LO https://repo1.maven.org/maven2/software/amazon/awssdk/bundle/2.31.9/bundle-2.31.9.jar  
  
# Add Kafka connector  
RUN cd /opt/flink/lib/ && \  
    curl -LO https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.4.0-1.20/flink-sql-connector-kafka-3.4.0-1.20.jar