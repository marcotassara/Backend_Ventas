FROM maven:3.9-eclipse-temurin-17-alpine AS compilacion

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline -q

COPY src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:17-jre-alpine AS ejecucion

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=compilacion /app/target/Springboot-API-REST-0.0.1-SNAPSHOT.jar app.jar

RUN chown appuser:appgroup app.jar

USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
