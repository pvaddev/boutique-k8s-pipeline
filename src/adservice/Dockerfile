# ---- build ----
FROM eclipse-temurin:21-jdk@sha256:3e3c176ffed168beb42c607be9bc1639b466cf00261a0fb04425562c9d0c5c2b AS build
WORKDIR /app
COPY gradlew .
COPY gradle/ gradle/
COPY build.gradle settings.gradle ./
RUN ./gradlew downloadRepos --no-daemon
COPY src/ src/
RUN ./gradlew installDist --no-daemon

# ---- runtime ----
FROM eclipse-temurin:21-jre@sha256:cff19e6215689161eb6162c11b86b0c60ddf802164f2eaf48d570f8fb79a36c5
WORKDIR /app
COPY --from=build /app/build/install/hipstershop /app
ENV PORT=9555
EXPOSE 9555
RUN groupadd -g 10001 app && useradd -u 10001 -g app -M -s /usr/sbin/nologin app
USER 10001
ENTRYPOINT ["/app/bin/AdService"]
