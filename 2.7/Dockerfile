FROM golang:latest AS dataplaneapi-builder

ENV DATAPLANE_MINOR 2.8.0
ENV DATAPLANE_URL https://github.com/haproxytech/dataplaneapi.git

RUN git clone "${DATAPLANE_URL}" "${GOPATH}/src/github.com/haproxytech/dataplaneapi"
RUN cd "${GOPATH}/src/github.com/haproxytech/dataplaneapi" && \
    git checkout "v${DATAPLANE_MINOR}" && \
    make build && \
    cp build/dataplaneapi /dataplaneapi

FROM alpine:3.17 AS openssl-builder

ENV OPENSSL_SHA256 4e23619f73688144f2cd808e9567504c3bf10cb67ad9551280530fd54ca913e1
ENV OPENSSL_URL https://github.com/quictls/openssl/archive/refs/tags/OpenSSL_1_1_1t-quic1.tar.gz

RUN apk add --no-cache curl build-base make autoconf automake gcc libc-dev linux-headers && \
    curl -sfSL "${OPENSSL_URL}" -o openssl.tar.gz && \
    mkdir -p /tmp/openssl && \
    tar -xzf openssl.tar.gz -C /tmp/openssl --strip-components=1 && \
    rm -f openssl.tar.gz && \
    cd /tmp/openssl && \
    ./config --libdir=lib --prefix=/opt/quictls && \
    make -j $(nproc) && \
    make install && \
    rm -rf /tmp/openssl

FROM alpine:3.17

MAINTAINER Dinko Korunic <dkorunic@haproxy.com>

LABEL Name HAProxy
LABEL Release Community Edition
LABEL Vendor HAProxy
LABEL Version 2.7.9
LABEL RUN /usr/bin/docker -d IMAGE

ENV HAPROXY_BRANCH 2.7
ENV HAPROXY_MINOR 2.7.9
ENV HAPROXY_SHA256 1c134e67a241543741426941e81ad04cdd80acb89928c2afbd0c71740f257d6b
ENV HAPROXY_SRC_URL http://www.haproxy.org/download

ENV DATAPLANE_MINOR 2.8.0
ENV DATAPLANE_SHA256 ec685350576898bb0a00d656f427f5da7c28043551a3e0050c1b5301547e235a
ENV DATAPLANE_URL https://github.com/haproxytech/dataplaneapi/releases/download

ENV HAPROXY_UID haproxy
ENV HAPROXY_GID haproxy

COPY --from=dataplaneapi-builder /dataplaneapi /usr/local/bin/dataplaneapi
COPY --from=openssl-builder /opt/quictls /opt/quictls

RUN apk add --no-cache ca-certificates && \
    apk add --no-cache --virtual build-deps gcc libc-dev \
    linux-headers lua5.4-dev make openssl openssl-dev pcre2-dev tar \
    zlib-dev curl shadow && \
    curl -sfSL "${HAPROXY_SRC_URL}/${HAPROXY_BRANCH}/src/haproxy-${HAPROXY_MINOR}.tar.gz" -o haproxy.tar.gz && \
    echo "$HAPROXY_SHA256 *haproxy.tar.gz" | sha256sum -c - && \
    groupadd "$HAPROXY_GID" && \
    useradd -g "$HAPROXY_GID" "$HAPROXY_UID" && \
    mkdir -p /tmp/haproxy && \
    tar -xzf haproxy.tar.gz -C /tmp/haproxy --strip-components=1 && \
    rm -f haproxy.tar.gz && \
    make -C /tmp/haproxy -j"$(nproc)" TARGET=linux-musl CPU=generic USE_PCRE2=1 USE_PCRE2_JIT=1 USE_OPENSSL=1 \
                            USE_TFO=1 USE_LINUX_TPROXY=1 USE_GETADDRINFO=1 \
                            USE_LUA=1 LUA_LIB=/usr/lib/lua5.4 LUA_INC=/usr/include/lua5.4 \
                            USE_PROMEX=1 USE_SLZ=1 \
                            SSL_INC=/opt/quictls/include SSL_LIB=/opt/quictls/lib USE_QUIC=1 \
                            LDFLAGS="-L/opt/quictls/lib -Wl,-rpath,/opt/quictls/lib" \
                            all && \
    make -C /tmp/haproxy TARGET=linux2628 install-bin install-man && \
    ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy && \
    mkdir -p /var/lib/haproxy && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /var/lib/haproxy && \
    mkdir -p /usr/local/etc/haproxy && \
    ln -s /usr/local/etc/haproxy /etc/haproxy && \
    cp -R /tmp/haproxy/examples/errorfiles /usr/local/etc/haproxy/errors && \
    rm -rf /tmp/haproxy && \
    chmod +x /usr/local/bin/dataplaneapi && \
    ln -s /usr/local/bin/dataplaneapi /usr/bin/dataplaneapi && \
    touch /usr/local/etc/haproxy/dataplaneapi.hcl && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /usr/local/etc/haproxy/dataplaneapi.hcl && \
    apk del build-deps && \
    apk add --no-cache openssl zlib lua5.4-libs pcre2 && \
    rm -f /var/cache/apk/* && \
    echo "/lib:/usr/local/lib:/usr/lib:/opt/quictls/lib" > "/etc/ld-musl-$(uname -m).path" && \
    mkdir -p /opt/quictls/ssl && \
    rm -rf /opt/quictls/ssl/certs && \
    ln -s /etc/ssl/certs /opt/quictls/ssl/certs

COPY haproxy.cfg /usr/local/etc/haproxy
COPY docker-entrypoint.sh /

STOPSIGNAL SIGUSR1

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["haproxy", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
