FROM docker.io/golang:1.17.0-alpine as builder

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG VERSION

ENV CGO_ENABLED=0 \
    GOPATH=/go \
    GOBIN=/go/bin \
    GO111MODULE=on

COPY ./apps/coredns/plugin.cfg /plugin.cfg

WORKDIR /go/src/coredns

# We apply cap_net_bind_service so that coredns can be run as
# non-root and still listen on port less than 1024

# hadolint ignore=DL4006
RUN \
  export GOOS \
  && GOOS=$(echo ${TARGETPLATFORM} | cut -d / -f1) \
  && export GOARCH \
  && GOARCH=$(echo ${TARGETPLATFORM} | cut -d / -f2) \
  && export GOARM \
  && GOARM=$(echo ${TARGETPLATFORM} | cut -d / -f3 | cut -c2-) \
  && apk --no-cache --no-progress add ca-certificates git libcap \
  && update-ca-certificates \
  && git clone https://github.com/coredns/coredns.git . \
  && git checkout v${VERSION}

RUN \
  mv /plugin.cfg . \
  && go get github.com/openshift/coredns-mdns \
  && go get github.com/coredns/alternate \
  && go get github.com/jwhited/wgsd \
  && go get github.com/miekg/coredns-git \
  && go get github.com/ori-edge/k8s_gateway \
  && go generate \
  && go mod tidy \
  && go build -ldflags "-s -w -X github.com/coredns/coredns/coremain.GitCommit=$(git describe --always)" -o coredns \
  && setcap cap_net_bind_service=+ep ./coredns

FROM gcr.io/distroless/static:nonroot

ARG VERSION

LABEL org.opencontainers.image.title="CoreDNS"
LABEL org.opencontainers.image.authors="Anthony Rabbito <hello@anthonyrabbito.com>"
LABEL org.opencontainers.image.version="${VERSION}"
LABEL coredns-external-plugins="alternate, mdns, wgsd, coredns-git, k8s_gateway"

COPY --from=builder /go/src/coredns/coredns /coredns
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

COPY ./apps/coredns/Corefile /config/Corefile

USER nonroot:nonroot

EXPOSE 53 53/udp

CMD ["/coredns", "-conf", "/config/Corefile"]

LABEL "maintainer"="Devin Buhl <devin.kray@gmail.com>, Bernd Schorgers <me@bjw-s.dev>"
LABEL "org.opencontainers.image.source"="https://github.com/k8s-at-home/container-images"
