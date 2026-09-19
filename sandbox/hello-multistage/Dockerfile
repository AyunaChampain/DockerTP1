FROM eclipse-temurin:21-jdk-alpine
WORKDIR /usr/src
COPY Main.java .
RUN javac Main.java

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=0 /usr/src/Main.class .
CMD ["java", "Main"]