FROM jenkins/inbound-agent:latest-jdk17

USER root

RUN apt-get update && apt-get install -y \
    build-essential \
    g++ \
    cmake \
    git \
    clang-tidy \
    clang-format \
    libgtest-dev \
    && apt-get clean

USER jenkins