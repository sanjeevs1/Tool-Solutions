# ========
# Stage 1: Base image including OS and key packages
# ========
ARG njobs
ARG bazel_mem

FROM ubuntu:18.04 AS tensorflow-base

RUN if ! [ "$(arch)" = "aarch64" ] ; then exit 1; fi

#Install core OS packages
RUN apt-get -y update && \
    apt-get -y install software-properties-common && \
    add-apt-repository ppa:ubuntu-toolchain-r/test && \
    apt-get -y install \
      autoconf \
      bc \
      build-essential \
      cmake \
      curl \
      g++-9 \
      gcc-9 \
      gettext-base \
      gfortran-9 \
      git \
      iputils-ping \
      libbz2-dev \
      libc++-dev \
      libcgal-dev \
      libffi-dev \
      libfreetype6-dev \
      libhdf5-dev \
      libjpeg-dev \
      liblzma-dev \
      libncurses5-dev \
      libncursesw5-dev \
      libpng-dev \
      libreadline-dev \
      libssl-dev \
      libsqlite3-dev \
      libxml2-dev \
      libxslt-dev \
      locales \
      moreutils \
      openjdk-8-jdk \
      openssl \
      python-openssl \
      rsync \
      ssh \
      sudo \
      time \
      unzip \
      vim \
      wget \
      xz-utils \
      zip \
      zlib1g-dev

# Make gcc 9 the default
RUN update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 1 --slave /usr/bin/g++ g++ /usr/bin/g++-9 && \
    update-alternatives --install /usr/bin/gfortran gfortran /usr/bin/gfortran-9 1

# DOCKER_USER for the Docker user
ENV DOCKER_USER=ubuntu

# Setup default user
RUN useradd --create-home -s /bin/bash -m $DOCKER_USER && echo "$DOCKER_USER:Arm2020" | chpasswd && adduser $DOCKER_USER sudo
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections

# Import profile for bash
COPY bash_profile /home/$DOCKER_USER/.bash_profile
RUN chown $DOCKER_USER:$DOCKER_USER /home/$DOCKER_USER/.bash_profile


# ========
# Stage 2: augment the base image with some essential libs
# ========
FROM tensorflow-base AS tensorflow-libs
ARG njobs
ARG dnnl_opt
ENV NP_MAKE="${njobs}" \
    DNNL_BUILD="${dnnl_opt}"

# Key version numbers
ENV PY_VERSION=3.7.0 \
    OPENBLAS_VERSION=0.3.7 \
    DNNL_VERSION=0.21.3

# Package build parameters
ENV PROD_DIR=/opt \
    PACKAGE_DIR=/packages

# Make directories to hold package source & build directories (PACKAGE_DIR)
# and install build directories (PROD_DIR).
RUN mkdir -p $PACKAGE_DIR && \
    mkdir -p $PROD_DIR

# Build Arm Optimized Routines from source
# provides optimised maths library fucntions for Aarch64
# see https://github.com/ARM-software/optimized-routines
COPY scripts/build-arm_opt_routines.sh $PACKAGE_DIR/.
COPY patches/config.mk $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-arm_opt_routines.sh

# Common compiler settings for remaining builds
# this ads arm_opt_routined into the LDFLAGS by default.
ENV CC=gcc \
    CXX=g++ \
    BASE_CFLAGS="-mcpu=native" \
    BASE_LDFLAGS="-L$PROD_DIR/arm_opt_routines/lib -lmathlib -lm" \
    LD_LIBRARY_PATH="$PROD_DIR/arm_opt_routines/lib"

# Build python from source
COPY scripts/build-cpython.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-cpython.sh

# Build OpenBLAS from source
COPY scripts/build-openblas.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-openblas.sh
ENV OPENBLAS_DIR=$PROD_DIR/openblas/$OPENBLAS_VERSION
ENV LD_LIBRARY_PATH=$OPENBLAS_DIR/lib:$LD_LIBRARY_PATH

# Build DNNL from source
COPY scripts/build-dnnl.sh $PACKAGE_DIR/.
# Patch for MKL-DNN, v0.20.6
COPY patches/mkldnn.patch $PACKAGE_DIR/mkldnn.patch
# Patch for DNNL  v1.1.2
#COPY patches/dnnl.patch $PACKAGE_DIR/dnnl.patch
RUN $PACKAGE_DIR/build-dnnl.sh
ENV DNNL_DIR=$PROD_DIR/dnnl/$DNNL_VERSION
ENV LD_LIBRARY_PATH=$DNNL_DIR/lib:$LD_LIBRARY_PATH

# ========
# Stage 3: install essential python dependencies into a venv
# ========
FROM tensorflow-libs AS tensorflow-tools
ARG njobs
ENV NP_MAKE="${njobs}"

# Key version numbers
ENV NUMPY_VERSION=1.17.1 \
    NPY_DISTUTILS_APPEND_FLAGS=1

# Using venv means this can be done in userspace
WORKDIR /home/$DOCKER_USER
USER $DOCKER_USER
ENV PACKAGE_DIR=/home/$DOCKER_USER/$PACKAGE_DIR
RUN mkdir -p $PACKAGE_DIR

# Setup a Python virtual environment
ENV VENV_DIR=/home/$DOCKER_USER/python3-venv
ENV VENV_ACT=$VENV_DIR/bin/activate
RUN python -m venv $VENV_DIR
ENV PATH="$VENV_DIR/bin:$PATH"

# Install some basic python packages
RUN pip install --no-cache-dir --upgrade pip
RUN pip install --no-cache-dir "setuptools>=41.0.0" six mock wheel cython

# Build numpy from source, using OpenBLAS for BLAS calls
COPY scripts/build-numpy.sh $PACKAGE_DIR/.
COPY patches/site.cfg $PACKAGE_DIR/site.cfg
RUN $PACKAGE_DIR/build-numpy.sh

# Install some TensorFlow essentials
RUN pip install --no-cache-dir keras_applications==1.0.8 --no-deps
RUN pip install --no-cache-dir keras_preprocessing==1.0.5 --no-deps
RUN HDF5_DIR=/usr/lib/aarch64-linux-gnu/hdf5/serial pip install h5py
RUN pip install --no-cache-dir ck absl-py pycocotools

# Install OpenCV into our venv,
COPY scripts/build-opencv.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-opencv.sh

CMD ["bash", "-l"]

# ========
# Stage 4: install Bazel, and build tensorflow
# ========
FROM tensorflow-libs AS tensorflow-dev
ARG njobs
ARG bazel_mem
ARG dnnl_opt
ENV NP_MAKE="${njobs}" \
    BZL_RAM="${bazel_mem}" \
    DNNL_BUILD="${dnnl_opt}"

# Key version numbers
ENV BZL_VERSION=0.29.1 \
    TF_VERSION=1.15.2

# Use a PACKAGE_DIR in userspace
WORKDIR /home/$DOCKER_USER
USER $DOCKER_USER
ENV PACKAGE_DIR=/home/$DOCKER_USER/$PACKAGE_DIR
RUN mkdir -p $PACKAGE_DIR

# Copy in the Python virtual environment
ENV VENV_DIR=/home/$DOCKER_USER/python3-venv
COPY --chown=$DOCKER_USER:$DOCKER_USER --from=tensorflow-tools $VENV_DIR /home/$DOCKER_USER/python3-venv
ENV PATH="$VENV_DIR/bin:$PATH"

# Build Bazel
COPY scripts/build-bazel.sh $PACKAGE_DIR/.
RUN $PACKAGE_DIR/build-bazel.sh
ENV PATH=$PACKAGE_DIR/bazel/output:$PATH

# Build TensorFlow
COPY scripts/build-tensorflow.sh $PACKAGE_DIR/.
COPY patches/tf_dnnl_decoupling.patch $PACKAGE_DIR/tf_dnnl_decoupling.patch
COPY patches/tensorflow.patch $PACKAGE_DIR/tensorflow.patch
RUN $PACKAGE_DIR/build-tensorflow.sh

CMD ["bash", "-l"]

# ========
# Stage 5: install tensorflow benchmarks
# ========
FROM tensorflow-libs AS tensorflow

WORKDIR /home/$DOCKER_USER
USER $DOCKER_USER

# Copy in the Python virtual environment
ENV VENV_DIR=/home/$DOCKER_USER/python3-venv
COPY --chown=$DOCKER_USER:$DOCKER_USER --from=tensorflow-dev $VENV_DIR /home/$DOCKER_USER/python3-venv
ENV PATH="$VENV_DIR/bin:$PATH"

# Clone Tensorflow benchmarks
RUN git clone https://github.com/tensorflow/benchmarks.git

CMD ["bash", "-l"]


