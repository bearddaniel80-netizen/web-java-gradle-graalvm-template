# syntax=docker/dockerfile:1.7
# ---------- deps ----------
FROM node:22-alpine AS deps

ENV GIT_BASE=github.com/bearddaniel80-netizen

RUN apk add --no-cache git

RUN git clone --depth=1 https://${GIT_BASE}/web-react-template.git

WORKDIR /build

RUN cp /web-react-template/frontend/package.json .

RUN --mount=type=cache,target=/root/.npm \
    npm install

# ---------- React build ----------
FROM deps AS react-runtime

WORKDIR /app

COPY --from=deps /build/node_modules ./node_modules

COPY --from=deps /web-react-template/frontend .
# To generate docs
# RUN npm run docs

RUN npm run build

# ------- assembly ---------------
FROM deps AS assembly

ARG DOMAIN
ARG APP_LANG
ARG BUILD_FILE
ARG EXT

ENV BUILD_FILE=${BUILD_FILE}

ENV APP_LANG=${APP_LANG}

ENV EXT=${EXT}

ENV DOMAIN=${DOMAIN}

ENV BUILD_DIR=/build-downloads
ENV COMMON_DIR=/common-downloads
ENV DOMAIN_DIR=/${DOMAIN}-downloads

ENV GIT_REPO=web-${APP_LANG}-lib

ENV GIT_URL=https://github.com/bearddaniel80-netizen/${GIT_REPO}.git

RUN git clone -b build --single-branch ${GIT_URL} ${BUILD_DIR}

RUN git clone -b common --single-branch ${GIT_URL} ${COMMON_DIR}

RUN git clone -b ${DOMAIN} --single-branch ${GIT_URL} ${DOMAIN_DIR}

# ........ build file ..........
WORKDIR /app/build

RUN cp ${BUILD_DIR}/${BUILD_FILE} .
RUN mv ${BUILD_FILE} build.gradle

# ........ actual code ..........
WORKDIR /app/${APP_LANG}/src/main/${APP_LANG}/example

RUN cp ${COMMON_DIR}/Application.${EXT} .

# ........ resources ..............
WORKDIR /app/${APP_LANG}/src/main

RUN cp -R ${DOMAIN_DIR}/resources .

# ---------- Gradle distribution ----------
# Reuse Gradle from the official image so we don't hardcode a version/URL
FROM gradle:8-jdk21 AS gradle-dist

# -------- build (GraalVM native image) ----------------
FROM ghcr.io/graalvm/native-image-community:21 AS build

ARG APP_LANG
ENV APP_LANG=${APP_LANG}

# Gradle's start script needs xargs (findutils); harmless if already present
RUN microdnf install -y findutils && microdnf clean all

COPY --from=gradle-dist /opt/gradle /opt/gradle
ENV GRADLE_HOME=/opt/gradle
ENV GRADLE_USER_HOME=/root/.gradle
ENV PATH="${GRADLE_HOME}/bin:${PATH}"

WORKDIR /app
COPY --from=assembly /app/build ./

RUN --mount=type=cache,target=/root/.gradle,sharing=locked \
    gradle --no-daemon dependencies > /dev/null

COPY ${APP_LANG}/ .

COPY --from=assembly /app/${APP_LANG} .

# Copy React production build into Spring Boot static resources
COPY --from=react-runtime \
    /app/dist/ ./src/main/resources/static/

# Remove -x test to run spring test
# Output binary: /app/build/native/nativeCompile/aql
RUN --mount=type=cache,target=/root/.gradle,sharing=locked \
    gradle --no-daemon nativeCompile -x test

# ---------- Native runtime ----------
# No JRE needed. Binary is dynamically linked against glibc, so use a glibc base.
FROM debian:bookworm-slim AS runtime
WORKDIR /app
COPY --from=build /app/build/native/nativeCompile/aql ./aql
ENTRYPOINT ["./aql"]