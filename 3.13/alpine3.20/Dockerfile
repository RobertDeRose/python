#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM alpine:3.20

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# runtime dependencies
RUN set -eux; \
	apk add --no-cache \
		ca-certificates \
		tzdata \
	;

ENV GPG_KEY 7169605F62C751356D054A26A821E680E5FA6305
ENV PYTHON_VERSION 3.13.0
ENV PYTHON_SHA256 086de5882e3cb310d4dca48457522e2e48018ecd43da9cdf827f6a0759efb07d

RUN set -eux; \
	\
	apk add --no-cache --virtual .build-deps \
		gnupg \
		tar \
		xz \
		\
		bluez-dev \
		bzip2-dev \
		dpkg-dev dpkg \
		expat-dev \
		findutils \
		gcc \
		gdbm-dev \
		libc-dev \
		libffi-dev \
		libnsl-dev \
		libtirpc-dev \
		linux-headers \
		make \
		ncurses-dev \
		openssl-dev \
		pax-utils \
		readline-dev \
		sqlite-dev \
		tcl-dev \
		tk \
		tk-dev \
		util-linux-dev \
		xz-dev \
		zlib-dev \
	; \
	\
	wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
	echo "$PYTHON_SHA256 *python.tar.xz" | sha256sum -c -; \
	wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc"; \
	GNUPGHOME="$(mktemp -d)"; export GNUPGHOME; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$GPG_KEY"; \
	gpg --batch --verify python.tar.xz.asc python.tar.xz; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME" python.tar.xz.asc; \
	mkdir -p /usr/src/python; \
	tar --extract --directory /usr/src/python --strip-components=1 --file python.tar.xz; \
	rm python.tar.xz; \
	\
	cd /usr/src/python; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	./configure \
		--build="$gnuArch" \
		--enable-loadable-sqlite-extensions \
		$(test "$gnuArch" != 'riscv64-linux-musl' && echo '--enable-optimizations') \
		--enable-option-checking=fatal \
		--enable-shared \
		--with-lto \
		--with-system-expat \
		--with-ensurepip \
	; \
	nproc="$(nproc)"; \
# set thread stack size to 1MB so we don't segfault before we hit sys.getrecursionlimit()
# https://github.com/alpinelinux/aports/commit/2026e1259422d4e0cf92391ca2d3844356c649d0
	EXTRA_CFLAGS="-DTHREAD_STACK_SIZE=0x100000"; \
	LDFLAGS="${LDFLAGS:--Wl},--strip-all"; \
# Workaround https://github.com/python/cpython/issues/125067
	printf '%s\n' \
		"diff --git a/Lib/test/test_xml_etree.py b/Lib/test/test_xml_etree.py" \
		"index ebec9d8f18a..385735c1e18 100644" \
		"--- a/Lib/test/test_xml_etree.py" \
		"+++ b/Lib/test/test_xml_etree.py" \
		"@@ -1504,9 +1504,11 @@ def test_simple_xml(self, chunk_size=None, flush=False):" \
		"         self.assert_event_tags(parser, [('end', 'root')])" \
		"         self.assertIsNone(parser.close())" \
		"" \
		"+    @unittest.skip('Work around: https://github.com/python/cpython/issues/125067')" \
		"     def test_simple_xml_chunk_1(self):" \
		"         self.test_simple_xml(chunk_size=1, flush=True)" \
		"" \
		"+    @unittest.skip('Work around: https://github.com/python/cpython/issues/125067')" \
		"     def test_simple_xml_chunk_5(self):" \
		"         self.test_simple_xml(chunk_size=5, flush=True)" \
		"" \
		"@@ -1731,6 +1733,7 @@ def test_flush_reparse_deferral_enabled(self):" \
		"" \
		"         self.assert_event_tags(parser, [('end', 'doc')])" \
		"" \
		"+    @unittest.skip('Work around: https://github.com/python/cpython/issues/125067')" \
		"     def test_flush_reparse_deferral_disabled(self):" \
		"         parser = ET.XMLPullParser(events=('start', 'end'))" > /tmp/python-3.13-libexpat-workaround.patch ; \
	patch -p1  </tmp/python-3.13-libexpat-workaround.patch || true ; \
	\
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:-}" \
	; \
# https://github.com/docker-library/python/issues/784
# prevent accidental usage of a system installed libpython of the same version
	rm python; \
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:--Wl},-rpath='\$\$ORIGIN/../lib'" \
		python \
	; \
	make install; \
	\
	cd /; \
	rm -rf /usr/src/python; \
	\
	find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test -o -name __pycache__ \) \) -o \
			\( -type f -a \( -name '*.py[co]' -o -name 'libpython*.a' -o -name *test*.so \) \) \
		\) -exec rm -rf '{}' + \
	; \
	\
	find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec scanelf --needed --nobanner --format '%n#p' '{}' ';' \
		| tr ',' '\n' \
		| sort -u \
		| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
		| xargs -rt apk add --no-network --virtual .python-rundeps \
	; \
	apk del --no-network .build-deps; \
	\
	export PYTHONDONTWRITEBYTECODE=1; \
	python3 --version; \
	pip3 --version

# make some useful symlinks that are expected to exist ("/usr/local/bin/python" and friends)
RUN set -eux; \
	for src in idle3 pip3 pydoc3 python3 python3-config; do \
		dst="$(echo "$src" | tr -d 3)"; \
		[ -s "/usr/local/bin/$src" ]; \
		[ ! -e "/usr/local/bin/$dst" ]; \
		ln -svT "$src" "/usr/local/bin/$dst"; \
	done

CMD ["python3"]
