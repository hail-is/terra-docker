FROM us.gcr.io/broad-dsp-gcr-public/terra-jupyter-gatk:2.0.7

USER root

# need to apt-get everything for python since we can only copy pip installed packages
RUN apt-get update && apt-get install -yq --no-install-recommends \
  jq \
  g++ \
  less \
  liblz4-dev \
  libmagick++-dev \
  iproute2 \
  # Nirvana/.NET Core dependencies \
  ca-certificates \
  libc6 \
  libgcc1 \
  libgssapi-krb5-2 \
  libicu60 \
  liblttng-ust0 \
  libssl1.0.0 \
  libstdc++6 \
  zlib1g \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*


# Spark setup.
# Copied from terra-jupyter-hail; keep updated.

# Note Spark and Hadoop are mounted from the outside Dataproc VM.
# Make empty conf dirs for the update-alternatives commands.
RUN mkdir -p /etc/spark/conf.dist && mkdir -p /etc/hadoop/conf.empty && mkdir -p /etc/hive/conf.dist \
    && update-alternatives --install /etc/spark/conf spark-conf /etc/spark/conf.dist 100 \
    && update-alternatives --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.empty 100 \
    && update-alternatives --install /etc/hive/conf hive-conf /etc/hive/conf.dist 100

ENV HAIL_VERSION=0.2.74
ENV PIP_USER=false

# For dataproc clusters, this path with will be automatically mounted. Else,
# this is effectively ignored. On GCE VMs, this will result in failures to
# import the pyspark package.
ENV PYTHONPATH $PYTHONPATH:/usr/lib/spark/python
ENV PYSPARK_PYTHON=python3

RUN pip3 install pypandoc \
      nbstripout \
      papermill \
      "git+https://github.com/all-of-us/workbench-snippets.git#egg=terra_widgets&subdirectory=py" \
    && pip3 install --no-dependencies hail==$HAIL_VERSION \
    && X=$(mktemp -d) \
    && mkdir -p $X \
    && (cd $X && pip3 download hail==$HAIL_VERSION --no-dependencies && \
        unzip hail*.whl &&  \
        grep 'Requires-Dist: ' hail*dist-info/METADATA | sed 's/Requires-Dist: //' | sed 's/ (//' | sed 's/)//' | grep -v 'pyspark' | xargs pip install) \
    && rm -rf $X

ENV PIP_USER=true

# Install Wondershaper from source, for client-side egress limiting.
RUN cd /usr/local/share && \
  git clone https://github.com/magnific0/wondershaper.git --depth 1 && \
  cd wondershaper && \
  make install && \
  cd $HOME

# Plink install
ENV PLINK_VERSION 20210416
RUN mkdir -p /tmp/plink && \
  cd /tmp/plink && \
  curl -L -o plink.zip "http://s3.amazonaws.com/plink1-assets/plink_linux_x86_64_${PLINK_VERSION}.zip" && \
  unzip plink.zip && \
  mv plink /bin/plink && \
  cd $HOME && \
  rm -rf /tmp/plink

RUN mkdir -p /tmp/plink2 && \
  cd /tmp/plink2 && \
  curl -L -o plink2.zip "https://s3.amazonaws.com/plink2-assets/alpha2/plink2_linux_x86_64.zip" && \
  unzip plink2.zip && \
  mv plink2 /bin/plink2 && \
  cd $HOME && \
  rm -rf /tmp/plink2

RUN mkdir -p /tmp/prsice && \
  cd /tmp/prsice && \
  curl -L -o prsice.zip https://github.com/choishingwan/PRSice/releases/download/2.3.5/PRSice_linux.zip && \
  unzip prsice.zip && \
  rm prsice.zip && \
  cd $HOME && \
  mv /tmp/prsice /opt/prsice && \
  ln -s "/opt/prsice/PRsice_linux" /bin/prsice

# BOLT-LMM install
ENV BOLT_LMM_VERSION v2.3.6
RUN mkdir -p /tmp/bolt-lmm && \
  cd /tmp/bolt-lmm && \
  curl -L -o bolt-lmm.tar.gz "https://storage.googleapis.com/broad-alkesgroup-public/BOLT-LMM/downloads/BOLT-LMM_${BOLT_LMM_VERSION}.tar.gz" && \
  tar -xvzf bolt-lmm.tar.gz && \
  mv "BOLT-LMM_${BOLT_LMM_VERSION}" /opt/bolt-lmm && \
  ln -s "/opt/bolt-lmm/bolt" /bin/bolt && \
  cd $HOME && \
  rm -rf /tmp/bolt-lmm

# REGENIE install
ENV REGENIE_VERSION v2.0.2
RUN mkdir -p /tmp/regenie && \
  cd /tmp/regenie && \
  curl -L -o regenie.zip "https://github.com/rgcgithub/regenie/releases/download/${REGENIE_VERSION}/regenie_${REGENIE_VERSION}.gz_x86_64_Linux.zip" && \
  unzip regenie.zip && \
  mv regenie_${REGENIE_VERSION}.gz_x86_64_Linux /bin/regenie && \
  cd $HOME && \
  rm -rf /tmp/regenie

# Install .NET Core + Nirvana
ENV DOTNET_VERSION=2.1.26
ENV NIRVANA_VERSION 3.14.0
ENV NIRVANA_ZIP_URL="https://github.com/Illumina/Nirvana/releases/download/v${NIRVANA_VERSION}/Nirvana-${NIRVANA_VERSION}-dotnet-2.1.0.zip"

# Based on https://github.com/MichaelStromberg-Illumina/DotNetMisc/blob/master/Dockerfiles/Nirvana/Dockerfile#L13
RUN curl -SL --output dotnet.tar.gz "https://dotnetcli.azureedge.net/dotnet/Runtime/${DOTNET_VERSION}/dotnet-runtime-${DOTNET_VERSION}-linux-x64.tar.gz" \
    && dotnet_sha512='41cc13f14dd7721a079bdd0ab489de40e9d4f32787239a26e7d10fcb0020a8e78d446c3b430b4bf80a557a925b3ca87d7981bfda4bbf9495cc44b1d42d877c40' \
    && echo "${dotnet_sha512} dotnet.tar.gz" | sha512sum -c - \
    && mkdir -p /usr/share/dotnet \
    && tar -zxf dotnet.tar.gz -C /usr/share/dotnet \
    && rm dotnet.tar.gz \
    && ln -s /usr/share/dotnet/dotnet /usr/bin/dotnet \
    && mkdir -p /opt/nirvana \
    && curl -SL --output nirvana.zip $NIRVANA_ZIP_URL \
    && unzip -d /opt/nirvana nirvana.zip \
    && rm nirvana.zip

ENV GCTA_VERSION=v1.94.0Beta
RUN mkdir -p /tmp/gcta && \
    cd /tmp/gcta && \
    curl -L -o gcta.zip "https://yanglab.westlake.edu.cn/software/gcta/bin/gcta_${GCTA_VERSION}_linux_kernel_4_x86_64.zip" && \
    unzip gcta.zip && \
    mv "gcta_${GCTA_VERSION}_linux_kernel_4_x86_64/gcta_${GCTA_VERSION}_linux_kernel_4_x86_64_static" /bin/gcta && \
    cd $HOME && \
    rm -rf /tmp/gcta

# Install bcftools
RUN mkdir -p /tmp/bcftools && \
    cd /tmp/bcftools && \
    curl -L -o bcftools.tar.bz2 https://github.com/samtools/bcftools/releases/download/1.12/bcftools-1.12.tar.bz2 && \
    bzip2 -d bcftools.tar.bz2 && \
    tar xvf bcftools.tar && \
    cd bcftools-1.12 && \
    make && \
    make install && \
    cd $HOME && \
    rm -rf /tmp/bcftools

# Install vcftools
RUN mkdir -p /tmp/vcftools && \
    cd /tmp/vcftools && \
    curl -L -o vcftools.tar.gz https://github.com/vcftools/vcftools/releases/download/v0.1.16/vcftools-0.1.16.tar.gz && \
    tar xvzf vcftools.tar.gz && \
    cd vcftools-0.1.16 && \
    ./configure && \
    make && \
    make install && \
    cd $HOME && \
    rm -rf /tmp/vcftools

# Install Nextflow
ENV NXF_MODE google
RUN mkdir -p /tmp/nextflow && \
  cd /tmp/nextflow && \
  curl -s https://get.nextflow.io | bash && \
  mv nextflow /bin/nextflow && \
  cd $HOME && \
  chmod 777 /bin/nextflow && \
  chown -R $USER:users $HOME/.nextflow && \
  rm -rf /tmp/nextflow

ENV USER jupyter
USER $USER

RUN R -e 'BiocManager::install(c( \
  "SAIGEgds", \
  "GENESIS"))'
