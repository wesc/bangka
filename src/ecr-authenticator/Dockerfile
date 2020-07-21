FROM docker:19.03

RUN apk --update --no-cache add bash python3 py3-pip && \
    pip --no-cache-dir install awscli==1.18.100

COPY ./ecr-auth /usr/bin/ecr-auth
