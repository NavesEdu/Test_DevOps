FROM jenkins/inbound-agent:alpine-jdk17

USER root

RUN apk update && apk add --no-cache \
    build-base \
    g++ \
    cmake \
    git \
    clang-tools \
    gtest-dev

USER jenkins