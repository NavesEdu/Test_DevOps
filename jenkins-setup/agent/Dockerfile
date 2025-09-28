FROM jenkins/inbound-agent:alpine-jdk17

USER root

RUN apk update && apk add --no-cache \
    build-base \
    g++ \
    cmake \
    git \
    clang-extra-tools \
    gtest-dev \
    curl

USER jenkins

COPY agent.sh /usr/local/bin/agent.sh
RUN chmod +x /usr/local/bin/agent.sh
ENTRYPOINT ["/usr/local/bin/agent.sh"]