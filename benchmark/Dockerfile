# Replication package example for Reprodcubility Engineering

# Copyright 2021, Wolfgang Mauerer <wolfgang.mauerer@othr.de>
# Copyright 2021, Stefanie Scherzinger <stefanie.scherzinger@uni-passau.de>
# Copyright 2021, Stefan Klessinger <stefanie.scherzinger@uni-passau.de>
# SPDX-License-Identifier: GPL-2.0-only

# Start off of a long-term maintained base distribution
FROM ubuntu:18.04

MAINTAINER Stefan Klessinger <stefan.klessinger@uni-passau.de>

ENV DEBIAN_FRONTEND noninteractive
ENV LANG="C.UTF-8"
ENV LC_ALL="C.UTF-8"

RUN apt update -qq
RUN apt-get install -y gpg-agent
RUN apt install -y --no-install-recommends software-properties-common dirmngr
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9
RUN add-apt-repository "deb https://cloud.r-project.org/bin/linux/ubuntu bionic-cran40/"
RUN add-apt-repository ppa:c2d4u.team/c2d4u4.0+

RUN apt update && apt install -y --no-install-recommends \
	biber \
	build-essential \
	ca-certificates \
	curl \
	file \
	git \
	libpng-dev \
	nano \
	openssh-client \
	r-base \
	r-cran-ggplot2 \
	r-cran-knitr \
	r-cran-tinytex \
	sudo \
	tcl-dev \
	time \
	wget

# We need to manually build the R tikzDevice package because there is no
# appropriate distributed version in our repository
RUN Rscript -e 'install.packages("tikzDevice", repos="https://cloud.r-project.org")'

RUN useradd -m -G sudo -s /bin/bash repro && echo "repro:repro" | chpasswd
USER repro
WORKDIR /home/repro

# Prepare directory structure
## git-repos/       - for external git repositories
## build/           - temporary directory for out-of-tree builds
## bin/             - for generated binary executables
RUN mkdir -p $HOME/git-repos $HOME/build $HOME/bin


# Obtain sqlite sources from a git repo, and check out _one specific, defined state_
# instead of working with a (changing) HEAD
# NOTE: We use an unofficial git mirror of sqlite to avoid working with fossil, which
# is a fairly unusual choice of tool, for the sake of simplicity in this tutorial.
WORKDIR /home/repro/git-repos
RUN git clone https://github.com/lfd/sqlite.git
WORKDIR /home/repro/git-repos/sqlite

# Purely technical: Construct manifest file required for building sqlite
# This is an artefact of the way how sqlite is built
RUN git rev-parse --git-dir >/dev/null
RUN echo $(git log -1 --format=format:%H) > manifest.uuid
RUN echo C $(cat manifest.uuid) > manifest
RUN git log -1 --format=format:%ci%n | sed 's/ [-+].*$//;s/ /T/;s/^/D /' >> manifest

RUN mkdir -p /home/repro/build/sqlite
WORKDIR /home/repro/build/sqlite

# Configure and build sqlite (NOTE: out-of-tree building is recommended practice
# to avoid inadvertent checkins of binary or other generated artefacts)
RUN ../../git-repos/sqlite/configure --prefix=$HOME
RUN make

# Build an interactive shell
RUN gcc shell.c sqlite3.c -lpthread -ldl -lm -o ~/bin/sqpolite

# Build the latency measurement tool
RUN gcc $HOME/git-repos/sqlite/src/latency.c -I. -I$HOME/git-repos/sqlite/src sqlite3.o \
                                             -o ~/bin/latency -lm -ldl -lpthread

# Build the plugin the the "pretty please" (pplease) function
RUN gcc -I. -fPIC -shared $HOME/git-repos/sqlite/ext/pplease.c -o pplease.so

# Make custom-built binaries in ~/bin binaries available via PATH
ENV PATH $PATH:/home/repro/bin

# Set up TPC-H data and queries for SQLite.
WORKDIR /home/repro/git-repos
RUN git clone --recursive https://github.com/lovasoa/TPCH-sqlite

WORKDIR /home/repro/git-repos/TPCH-sqlite
RUN git checkout -b repro 23e420d8d49a6

COPY patches/TPCH-sqlite.diff /tmp

RUN mkdir -p /home/repro/queries.polite
RUN mkdir -p /home/repro/queries.impolite
COPY queries.polite/* /home/repro/queries.polite/
COPY queries.impolite/* /home/repro/queries.impolite/

# Generate self-contained measurement package that can
# be deployed on the target platform.
WORKDIR /home/repro/git-repos/TPCH-sqlite
RUN git archive --format=tar --prefix=TPCH-sqlite/ HEAD > /tmp/tpch.tar
WORKDIR /home/repro/git-repos/TPCH-sqlite/tpch-dbgen
RUN git archive --format=tar --prefix=TPCH-sqlite/tpch-dbgen/ HEAD > /tmp/tpch-dbgen.tar

WORKDIR /home/repro
RUN tar xf /tmp/tpch.tar
RUN tar xf /tmp/tpch-dbgen.tar

# Apply local fixes to components
WORKDIR /home/repro/TPCH-sqlite
RUN patch -p0 create_db.sh /tmp/TPCH-sqlite.diff 

WORKDIR /home/repro
COPY --chown=repro:repro scripts/dispatch.sh .
COPY --chown=repro:repro scripts/doall.sh .
COPY --chown=repro:repro scripts/prepare_data.sh .
COPY --chown=repro:repro clessbench .

RUN tar --transform 's,^,measure/,' -cjhf deliverable.tar.bz2 queries.*/ TPCH-sqlite/ bin/ \
    doall.sh dispatch.sh prepare_data.sh

WORKDIR /home/repro
COPY --chown=repro:repro results /home/repro/results
RUN curl -sL "https://yihui.org/tinytex/install-bin-unix.sh" | sh
