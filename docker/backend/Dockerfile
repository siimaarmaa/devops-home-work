# Võtab ehitatud jar-i ja paneb selle kergesse JRE konteinerisse.
# Kasutab mitte-root kasutajat turvalisuse pärast.

FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
WORKDIR /app
COPY target/spring-petclinic-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
