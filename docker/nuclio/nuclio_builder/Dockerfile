FROM gcr.io/cloud-builders/docker

ENV NUCLIO_VERSION=1.6.26
ENV NUCLIO_LABEL=1.6.26
ENV NUCLIO_DOCKER_REPO=local

RUN git clone https://github.com/nuclio/nuclio.git /src/nuclio/ \
    && cd /src/nuclio/ \
    && git checkout $NUCLIO_VERSION

RUN apt-get update && apt-get install make golang -y

COPY patch /src/nuclio

WORKDIR /src/nuclio

RUN patch -p1 < patch

COPY docker-entrypoint.sh /entrypoint.sh

COPY scripts /scripts

ENTRYPOINT ["/entrypoint.sh"]
