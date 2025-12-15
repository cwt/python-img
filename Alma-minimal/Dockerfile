FROM almalinux:10 AS builder

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# runtime dependencies
RUN set -eux; \
	dnf update -y \
	&& dnf install -y --setopt=install_weak_deps=False \
		ca-certificates \
		tzdata \
	&& dnf clean all

ENV PYTHON_VERSION 3.14.2
ENV PYTHON_SHA256 ce543ab854bc256b61b71e9b27f831ffd1bfd60a479d639f8be7f9757cf573e9

# Create symlink from lib64 to lib if lib64 exists, to ensure compatibility
RUN if [ -d "/usr/local/lib64" ]; then \
        rm -rf /usr/local/lib64 && ln -sf /usr/local/lib /usr/local/lib64; \
    fi

RUN set -eux; \
	\
	dnf install -y --setopt=install_weak_deps=False --setopt=tsflags=nodocs \
		bzip2-devel \
		expat-devel \
		findutils \
		gcc \
		gdbm-devel \
		glibc-devel \
		libffi-devel \
		libuuid-devel \
		libzstd-devel \
		make \
		mpdecimal-devel \
		ncurses-devel \
		openssl-devel \
		readline-devel \
		redhat-rpm-config \
		sqlite-devel \
		tcl-devel \
		tk-devel \
		xz-devel \
		zlib-devel \
		wget \
	&& dnf clean all \
	; \
	\
	wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
	echo "$PYTHON_SHA256 *python.tar.xz" | sha256sum -c -; \
	mkdir -p /usr/src/python; \
	tar --extract --directory /usr/src/python --strip-components=1 --file python.tar.xz; \
	rm python.tar.xz; \
	\
	cd /usr/src/python; \
	./configure \
		--enable-loadable-sqlite-extensions \
		--enable-option-checking=fatal \
		--enable-shared \
		--enable-optimizations \
		--with-ensurepip \
		--with-lto \
		--with-computed-gotos=yes \
		--prefix=/usr/local \
		--exec-prefix=/usr/local \
	; \
	nproc="$(nproc)"; \
# set thread stack size to 1MB so we don't segfault before we hit sys.getrecursionlimit()
# https://github.com/alpinelinux/aports/commit/2026e1259422d4e0cf92391ca2d3844356c649d0
	EXTRA_CFLAGS="-DTHREAD_STACK_SIZE=0x100000"; \
	LDFLAGS="${LDFLAGS:--Wl},--strip-all"; \
	arch="$(uname -m)"; \
# https://docs.python.org/3.12/howto/perf_profiling.html
# https://github.com/docker-library/python/pull/1000#issuecomment-2597021615
	case "$arch" in \
		x86_64|aarch64|riscv64) \
			# only add "-mno-omit-leaf" on arches that support it
			# https://gcc.gnu.org/onlinedocs/gcc-14.2.0/gcc/x86-Options.html#index-momit-leaf-frame-pointer-2
			# https://gcc.gnu.org/onlinedocs/gcc-14.2.0/gcc/AArch64-Options.html#index-momit-leaf-frame-pointer
			# RISC-V 64-bit also supports these flags
			EXTRA_CFLAGS="${EXTRA_CFLAGS:-} -fno-omit-frame-pointer -mno-omit-leaf-frame-pointer"; \
			;; \
		x86) \
			# don't enable frame-pointers on 32bit x86 due to performance drop.
			;; \
		*) \
			# other arches don't support "-mno-omit-leaf"
			EXTRA_CFLAGS="${EXTRA_CFLAGS:-} -fno-omit-frame-pointer"; \
			;; \
	esac; \
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
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
		\) -exec rm -rf '{}' + \
	; \
	\
# delete unversioned library to avoid conflict with base image libpython3.so
	rm -f /usr/local/lib/libpython3.so; \
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

FROM almalinux:10-minimal

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH
ENV LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib64:$LD_LIBRARY_PATH

# Copy Python installation from builder
COPY --from=builder /usr/local /usr/local

# Create symlink from lib64 to lib if lib64 exists, to ensure compatibility
RUN if [ -d "/usr/local/lib64" ]; then \
        rm -rf /usr/local/lib64 && ln -sf /usr/local/lib /usr/local/lib64; \
    fi

# runtime dependencies
RUN set -eux; \
	microdnf update -y \
	&& microdnf install -y --setopt=install_weak_deps=0 --setopt=tsflags=nodocs \
		ca-certificates \
		tzdata \
		expat \
		gdbm \
		libffi \
		libuuid \
		libzstd \
		mpdecimal \
		ncurses \
		openssl \
		readline \
		sqlite \
		tcl \
		tk \
		zlib \
	&& microdnf clean all

CMD ["python3"]
