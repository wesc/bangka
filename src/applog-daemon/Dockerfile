FROM alpine:3.11

RUN apk --no-cache --update add bash findutils runit
RUN mkdir -p /service/applog-cleanup /logs

COPY ./applog-cleanup.run /service/applog-cleanup/run
COPY . /src

CMD runsvdir /service
