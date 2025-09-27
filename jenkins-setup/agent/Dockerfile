FROM jenkins/inbound-agent:alpine-jdk17 # Imagem base muito menor

USER root

RUN apk update && apk add --no-cache \
    build-base \
    g++ \
    cmake \
    git \
    clang-tools \
    gtest-dev

USER jenkins