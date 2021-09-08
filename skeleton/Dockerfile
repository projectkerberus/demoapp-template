FROM maven:3.6.0-jdk-11-slim AS build
COPY src /home/app/src
COPY pom.xml /home/app
RUN mvn -f /home/app/pom.xml clean package


FROM bitnami/tomcat:9.0
COPY --from=build /home/app/target/*.war /opt/bitnami/tomcat/webapps_default/