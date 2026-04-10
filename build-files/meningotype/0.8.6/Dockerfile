ARG MENINGOTYPE_VER="0.8.6"
ARG ISPCR_VER="33"
ARG MLST_VER="2.33.0"
ARG ANY2FASTA_VER="0.8.1"
ARG BLAST_VER="2.17.0"

## Builder ##
FROM ubuntu:noble AS builder

WORKDIR /usr/local/bin

ARG MENINGOTYPE_VER
ARG ISPCR_VER
ARG ANY2FASTA_VER
ARG MLST_VER
ARG BLAST_VER
ARG MACHTYPE="x86_64-pc-linux-gnu"


RUN apt-get update && apt-get install -y \
    build-essential \
    wget \
    git \
    ca-certificates \
    libmoo-perl \
    liblist-moreutils-perl \
    libfile-which-perl \
    libjson-perl \
    libpath-tiny-perl \
    gzip \
    file \
    ncbi-blast+ \
    curl \
    parallel \
    procps \
    unzip && \
    apt-get autoclean && rm -rf /var/lib/apt/lists/*


# BLAST installation >= 2.16.0 as per MLST specifications.
RUN wget -q https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/${BLAST_VER}/ncbi-blast-${BLAST_VER}+-x64-linux.tar.gz && \
    tar xzf ncbi-blast-${BLAST_VER}+-x64-linux.tar.gz && \
    mv ncbi-blast-${BLAST_VER}+/bin/* /usr/local/bin/ && \
    rm -rf ncbi-blast-${BLAST_VER}+*

# ANY2FASTA installation >= 0.6.0 as per MLST specifications.
RUN wget -q https://github.com/tseemann/any2fasta/archive/refs/tags/v${ANY2FASTA_VER}.tar.gz && \
    tar xzf v${ANY2FASTA_VER}.tar.gz && \
    rm v${ANY2FASTA_VER}.tar.gz && \
    chmod +x any2fasta-${ANY2FASTA_VER}/any2fasta && \
    mv -v any2fasta-${ANY2FASTA_VER}/any2fasta /usr/local/bin && \
    rm -rf any2fasta-${ANY2FASTA_VER}

# MLST installation along with execution permissions for bin and scripts folders.
RUN git clone --branch v${MLST_VER} --depth 1 https://github.com/tseemann/mlst.git && \
    chmod a+x mlst/bin/* mlst/scripts/* && \
    mv mlst /opt/mlst

# isPcr v33 installation.
RUN wget https://hgwdev.gi.ucsc.edu/~kent/src/isPcr${ISPCR_VER}.zip && \
    unzip isPcr${ISPCR_VER}.zip && \
    cd isPcrSrc && \
    sed -i 's/-Werror//g' inc/common.mk && \
    mkdir -p bin/$MACHTYPE && \
    mkdir -p lib/$MACHTYPE && \
    make HOME=$PWD MACHTYPE=${MACHTYPE} C_INCLUDE_PATH=/usr/include LIBRARY_PATH=/usr/lib CFLAGS="-fcommon" && \
    mv bin/$MACHTYPE/* /usr/local/bin/ && \
    cd /usr/local/bin && \
    rm -rf isPcrSrc isPcr${ISPCR_VER}.zip


RUN git clone --branch v${MENINGOTYPE_VER} --depth 1 https://github.com/MDU-PHL/meningotype.git /opt/meningotype

COPY meningotype.py /opt/meningotype/meningotype/meningotype.py
COPY run_blast.py /opt/meningotype/meningotype/run_blast.py

FROM python:3.12.12-slim AS app

ARG MENINGOTYPE_VER

# 'LABEL' instructions tag the image with metadata that might be important to the user
LABEL base.image="python:3.12.12-slim"
LABEL dockerfile.version="1"
LABEL software="meningotype"
LABEL software.version="${MENINGOTYPE_VER}"
LABEL description="In silico typing of Neisseria meningitidis contigs"
LABEL website="https://github.com/MDU-PHL/meningotype"
LABEL license="https://github.com/MDU-PHL/meningotype/blob/master/LICENSE"
LABEL maintainer="Erin Young"
LABEL maintainer.email="eriny@utah.gov"
LABEL maintainer2="Raheel Ahmed"
LABEL maintainer2.email="raheelsyedahmed@gmail.com"

# perl is a core requirement for mlst and any2fasta, and the compression libraries appear to be a dependency of any2fasta.
RUN apt-get update && apt-get install -y --no-install-recommends \
    wget \
    ca-certificates \
    libmoo-perl \
    liblist-moreutils-perl \
    libfile-which-perl \
    libjson-perl \
    libpath-tiny-perl \
    gzip \
    bzip2 \
    unzip \
    curl \
    parallel \
    libgomp1 \
    procps && \
    apt-get autoclean && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/bin/* /usr/local/bin/
COPY --from=builder /opt/mlst /usr/local/bin/mlst
COPY --from=builder /opt/meningotype /opt/meningotype


ENV PATH="/usr/local/bin/mlst/bin:/usr/local/bin/mlst/scripts:$PATH" LC_ALL=C.UTF-8

RUN pip install /opt/meningotype

WORKDIR /data

RUN meningotype -h
RUN meningotype --version

# update the database
RUN meningotype --updatedb

CMD [ "meningotype", "--help" ]

## Test ##
FROM app AS test

# set working directory so that all test inputs & outputs are kept in /test
WORKDIR /test

# print help and version info; check dependencies (not all software has these options available)
# Mostly this ensures the tool of choice is in path and is executable
RUN meningotype --help && \
    meningotype --version 

# Check dependencies of meningotype
RUN meningotype --checkdeps

# Run the program's internal tests if available
RUN meningotype --test && \
    meningotype --test --finetype && \
    meningotype --test --all

# Option 2: write below common usage cases
RUN wget -q https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/022/869/645/GCA_022869645.1_ASM2286964v1/GCA_022869645.1_ASM2286964v1_genomic.fna.gz && \
    gzip -d GCA_022869645.1_ASM2286964v1_genomic.fna.gz && \
    meningotype GCA_022869645.1_ASM2286964v1_genomic.fna > results.txt && \
    head results.txt