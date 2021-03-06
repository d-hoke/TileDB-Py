FROM quay.io/pypa/manylinux1_x86_64

###############################################
# version args

ARG LIBTILEDB_VERSION=2.0.7
ENV LIBTILEDB_VERSION=$LIBTILEDB_VERSION

ARG LIBTILEDB_SHA=2058d3de138d6b11446c485fcfd0da70bfd6d038
ENV LIBTILEDB_SHA=$LIBTILEDB_SHA

ARG TILEDBPY_VERSION=0.6.6
ENV TILEDBPY_VERSION=$TILEDBPY_VERSION

###############################################
# python settings
# NOTE: MUST USE the 'mu' variant here to be compatible
#       with ~most~ linux distros (see manylinux README)
ENV PYTHON_BASE /opt/python/cp27-cp27mu/bin/

RUN useradd tiledb
ENV HOME /home/tiledb

# dependencies:
# - cmake (need recent) and auditwheel from pip
# - perl 5.10.0 for openssl
RUN  $PYTHON_BASE/pip install cmake==3.13.3 auditwheel && \
  curl -L https://install.perlbrew.pl | bash && \
  source $HOME/perl5/perlbrew/etc/bashrc && \
  perlbrew --notest install perl-5.10.0

ENV CMAKE /opt/python/cp27-cp27mu/bin/cmake

###############################################
# 1) Nothing builds under GCC 4.8 due to default constructor unused-parameter warnings
# 2) adding -lrt as a work-around for now because python2.7 doesn't link it, but it
#    ends up as an unlinked dependency.
# 3) Capnproto (TileDB Serialization) requeries -DKJ_USE_EPOLL=0 -D__BIONIC__=1 per
#    https://github.com/capnproto/capnproto/issues/350#issuecomment-270930594

ENV CXXFLAGS -Wno-unused-parameter -lrt -DKJ_USE_EPOLL=0 -D__BIONIC__=1
ENV CFLAGS -Wno-unused-parameter -lrt -DKJ_USE_EPOLL=0 -D__BIONIC__=1

# build libtiledb (core)
# notes:
#    1) we are using auditwheel from https://github.com/pypa/auditwheel
#       this verifies and tags wheel products with the manylinux1 label,
#       and allows us to build libtiledb once, install it to a normal
#       system path, and then use it to build wheels for all of the python
#       versions.
#    2) perl-5.10.0, built above, is required to build OpenSSL
RUN cd /home/tiledb/ && \
  source $HOME/perl5/perlbrew/etc/bashrc && \
  perlbrew use perl-5.10.0 && \
  git clone https://github.com/TileDB-Inc/TileDB && \
  git -C TileDB checkout $LIBTILEDB_SHA && \
  mkdir build && \
  cd build && \
  $CMAKE -DTILEDB_S3=ON -DTILEDB_CPP_API=ON -DTILEDB_HDFS=ON -DTILEDB_TESTS=OFF \
         -DTILEDB_SERIALIZATION=ON -DTILEDB_FORCE_ALL_DEPS:BOOL=ON \
         -DTILEDB_LOG_OUTPUT_ON_FAILURE:BOOL=ON \
         -DSANITIZER="OFF;-DCOMPILER_SUPPORTS_AVX2:BOOL=FALSE" \
         ../TileDB && \
  make -j8 && \
  make install-tiledb

ADD misc/pypi_linux/build.sh /usr/bin/build.sh
RUN chmod +x /usr/bin/build.sh

# add source directory as optional TILEDB_PY_REPO
ADD . /opt/TileDB-Py
