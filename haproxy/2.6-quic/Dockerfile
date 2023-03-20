# This Dockerfile combines the following code to create a haproxy build that emulates the official docker image with QUIC/HTTP3 support
# From: https://github.com/docker-library/haproxy/blob/f4ae2c3536eef4ca86d9b2d96bb97b1fb9d82f33/2.6/Dockerfile
# From: https://github.com/haproxytechblog/haproxy-2.6-http3/blob/main/init.sh

FROM debian:bullseye-slim

# roughly, https://salsa.debian.org/haproxy-team/haproxy/-/blob/732b97ae286906dea19ab5744cf9cf97c364ac1d/debian/haproxy.postinst#L5-6
RUN set -eux; \
	groupadd --gid 99 --system haproxy; \
	useradd \
		--gid haproxy \
		--home-dir /var/lib/haproxy \
		--no-create-home \
		--system \
		--uid 99 \
		haproxy \
	; \
	mkdir /var/lib/haproxy; \
	chown haproxy:haproxy /var/lib/haproxy

ENV OPENSSL_URL https://github.com/quictls/openssl/archive/refs/tags/openssl-3.0.8-quic1.tar.gz

ENV HAPROXY_VERSION 2.6.10
ENV HAPROXY_URL https://www.haproxy.org/download/2.6/src/haproxy-2.6.10.tar.gz
ENV HAPROXY_SHA256 e71b2cd9ca1043345f083a5225078ccf824dced2b5779d86f11fa4e88f451773

# see https://sources.debian.net/src/haproxy/jessie/debian/rules/ for some helpful navigation of the possible "make" arguments
RUN set -eux; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update && apt-get install -y --no-install-recommends \
		ca-certificates \
		gcc \
		libc6-dev \
		liblua5.3-dev \
		libpcre2-dev \
		libssl-dev \
		make \
		perl \
		wget \
	; \
	rm -rf /var/lib/apt/lists/*; \
	\
	wget -O openssl.tar.gz "$OPENSSL_URL"; \
	mkdir -p /usr/src/openssl; \
	tar -xzf openssl.tar.gz -C /usr/src/openssl --strip-components=1; \
	rm openssl.tar.gz; \
	cd /usr/src/openssl; \
	./Configure --libdir=lib --prefix=/usr/local/ssl; \
	make; \
	make install; \
	echo /usr/local/ssl/lib | tee -a /etc/ld.so.conf; \
	ldconfig; \
	cd -; \
	\
	wget -O haproxy.tar.gz "$HAPROXY_URL"; \
	echo "$HAPROXY_SHA256 *haproxy.tar.gz" | sha256sum -c; \
	mkdir -p /usr/src/haproxy; \
	tar -xzf haproxy.tar.gz -C /usr/src/haproxy --strip-components=1; \
	rm haproxy.tar.gz; \
	\
	makeOpts=' \
		TARGET=linux-glibc \
		USE_GETADDRINFO=1 \
		USE_LUA=1 LUA_INC=/usr/include/lua5.3 \
		USE_OPENSSL=1 \
		USE_PCRE2=1 USE_PCRE2_JIT=1 \
		USE_PROMEX=1 \
		USE_QUIC=1 \
		SSL_INC=/usr/local/ssl/include \
		SSL_LIB=/usr/local/ssl/lib \
		LDFLAGS="-Wl,-rpath,/usr/local/ssl/lib" \
		\
		EXTRA_OBJS=" \
		" \
	'; \
# https://salsa.debian.org/haproxy-team/haproxy/-/commit/53988af3d006ebcbf2c941e34121859fd6379c70
	dpkgArch="$(dpkg --print-architecture)"; \
	case "$dpkgArch" in \
		armel) makeOpts="$makeOpts ADDLIB=-latomic" ;; \
	esac; \
	\
	nproc="$(nproc)"; \
	eval "make -C /usr/src/haproxy -j '$nproc' all $makeOpts"; \
	eval "make -C /usr/src/haproxy install-bin $makeOpts"; \
	\
	mkdir -p /usr/local/etc/haproxy; \
	cp -R /usr/src/haproxy/examples/errorfiles /usr/local/etc/haproxy/errors; \
	rm -rf /usr/src/haproxy; \
	rm -rf /usr/src/openssl; \
	\
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
	find /usr/local -type f -executable -exec ldd '{}' ';' \
		| awk '/=>/ { print $(NF-1) }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	\
# smoke test
	haproxy -v

# https://www.haproxy.org/download/1.8/doc/management.txt
# "4. Stopping and restarting HAProxy"
# "when the SIGTERM signal is sent to the haproxy process, it immediately quits and all established connections are closed"
# "graceful stop is triggered when the SIGUSR1 signal is sent to the haproxy process"
STOPSIGNAL SIGUSR1

COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

USER haproxy

# no WORKDIR for backwards compatibility (to try to avoid breaking existing users)

CMD ["haproxy", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
