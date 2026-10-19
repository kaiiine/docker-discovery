FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /usr/src

COPY Main.java .

RUN javac Main.java



FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=build /usr/src/Main.class .

CMD ["java", "Main"]
