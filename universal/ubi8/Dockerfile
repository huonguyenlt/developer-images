# syntax=docker/dockerfile:1.3-labs

# updateBaseImages.sh can't operate on SHA-based tags as they're not date-based or semver-sequential, and therefore cannot be ordered
FROM quay.io/devfile/base-developer-image:ubi8-latest
LABEL maintainer="Red Hat, Inc."

LABEL com.redhat.component="devfile-universal-container"
LABEL name="devfile/universal-developer-image"
LABEL version="ubi8"

#label for EULA
LABEL com.redhat.license_terms="https://www.redhat.com/en/about/red-hat-end-user-license-agreements#UBI"

#labels for container catalog
LABEL summary="devfile universal developer image"
LABEL description="Image with developers tools. Languages SDK and runtimes included."
LABEL io.k8s.display-name="devfile-developer-universal"
LABEL io.openshift.expose-services=""

USER 0

# $PROFILE_EXT contains all additions made to the bash environment
ENV PROFILE_EXT=/etc/profile.d/udi_environment.sh
RUN touch ${PROFILE_EXT} & chown 10001 ${PROFILE_EXT}

USER 10001

# We install everything to /home/tooling/ as /home/user/ may get overriden, see github.com/eclipse/che/issues/22412
ENV HOME=/home/tooling

# Java
RUN curl -fsSL "https://get.sdkman.io/?rcupdate=false" | bash \
    && bash -c ". /home/tooling/.sdkman/bin/sdkman-init.sh \
             && sed -i "s/sdkman_auto_answer=false/sdkman_auto_answer=true/g" /home/tooling/.sdkman/etc/config \
	     && sed -i "s/sdkman_auto_env=false/sdkman_auto_env=true/g" /home/tooling/.sdkman/etc/config \
             && sdk install java 8.0.402-tem \
             && sdk install gradle \
             && sdk install maven \
             && sdk install jbang \
             && sdk flush archives \
             && sdk flush temp" \
         && chgrp -R 0 /home/tooling && chmod -R g=u /home/tooling

# Java-related environment variables are described and set by ${PROFILE_EXT}, which will be loaded by ~/.bashrc
# To make Java working for dash and other shells, it needs to initialize them in the Dockerfile.
ENV SDKMAN_CANDIDATES_API="https://api.sdkman.io/2"
ENV SDKMAN_CANDIDATES_DIR="/home/tooling/.sdkman/candidates"
ENV SDKMAN_DIR="/home/tooling/.sdkman"
ENV SDKMAN_PLATFORM="linuxx64"
ENV SDKMAN_VERSION="5.18.2"

ENV GRADLE_HOME="/home/tooling/.sdkman/candidates/gradle/current"
ENV JAVA_HOME="/home/tooling/.sdkman/candidates/java/8.0.402-tem"
ENV MAVEN_HOME="/home/tooling/.sdkman/candidates/maven/current"

ENV GRAALVM_HOME=/home/tooling/.sdkman/candidates/java/23.1.2.r21-mandrel

ENV PATH="/home/tooling/.krew/bin:$PATH"
ENV PATH="/home/tooling/.sdkman/candidates/maven/current/bin:$PATH"
ENV PATH="/home/tooling/.sdkman/candidates/java/current/bin:$PATH"
ENV PATH="/home/tooling/.sdkman/candidates/gradle/current/bin:$PATH"
ENV PATH="/home/tooling/.local/share/coursier/bin:$PATH"

# Config directories
RUN mkdir -p /home/tooling/.m2 && \
    mkdir -p /home/tooling/.gradle && \
    mkdir -p /home/tooling/.config/pip && \
    mkdir -p /home/tooling/.sbt/1.0 && \
    mkdir -p /home/tooling/.cargo && \
    mkdir -p /home/tooling/certs && \
    mkdir -p /home/tooling/.composer && \
    mkdir -p /home/tooling/.nuget && \
    chgrp -R 0 /home/tooling && chmod -R g=u /home/tooling

# Cloud
 
RUN echo 'alias docker=podman' >> ${PROFILE_EXT}

# Configure container engine
COPY --chown=0:0 containers.conf /etc/containers/containers.conf

## shellcheck
RUN <<EOF
dnf install -y xz
set -euf -o pipefail
TEMP_DIR="$(mktemp -d)"
cd "${TEMP_DIR}"
SHELL_CHECK_VERSION="0.8.0"
SHELL_CHECK_ARCH="x86_64"
SHELL_CHECK_TGZ="shellcheck-v${SHELL_CHECK_VERSION}.linux.${SHELL_CHECK_ARCH}.tar.xz"
SHELL_CHECK_TGZ_URL="https://github.com/koalaman/shellcheck/releases/download/v${SHELL_CHECK_VERSION}/${SHELL_CHECK_TGZ}"
curl -sSLO "${SHELL_CHECK_TGZ_URL}"
tar -xv --no-same-owner -f "${SHELL_CHECK_TGZ}"
mv "${TEMP_DIR}"/shellcheck-v${SHELL_CHECK_VERSION}/shellcheck /bin/shellcheck
cd -
rm -rf "${TEMP_DIR}"
EOF

# e2fsprogs setup
# Since e2fsprogs-static package has removed RHEL 8 distribution, it is not possible to install from the repository
# https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/considerations_in_adopting_rhel_8/index#removed-packages_changes-to-packages
RUN <<EOF
set -euf -o pipefail
TEMP_DIR="$(mktemp -d)"
cd "${TEMP_DIR}"
E2FSPROGS_VERSION="1.46.5"
E2FSPROGS_TGZ="e2fsprogs-${E2FSPROGS_VERSION}.tar.gz"
E2FSPROGS_TGZ_URL="https://mirrors.edge.kernel.org/pub/linux/kernel/people/tytso/e2fsprogs/v${E2FSPROGS_VERSION}/${E2FSPROGS_TGZ}"
E2FSPROGS_CHEKSUMS_URL="https://mirrors.edge.kernel.org/pub/linux/kernel/people/tytso/e2fsprogs/v${E2FSPROGS_VERSION}/sha256sums.asc"
curl -sSLO "${E2FSPROGS_TGZ_URL}"
curl -sSLO "${E2FSPROGS_CHEKSUMS_URL}"
sha256sum --ignore-missing -c "sha256sums.asc" 2>&1 | grep OK
tar -zxv --no-same-owner -f "${E2FSPROGS_TGZ}"
cd "e2fsprogs-${E2FSPROGS_VERSION}"
mkdir build
cd build
../configure --prefix=/usr --with-root-prefix="" --enable-elf-shlibs --disable-evms
make
make install
make install-libs
cd -
rm -rf "${TEMP_DIR}"
EOF

# Bash completions
RUN dnf -y install bash-completion \
    && dnf clean all \
    && rm -rf /var/cache/yum

RUN <<EOF
oc completion bash > /usr/share/bash-completion/completions/oc
tkn completion bash > /usr/share/bash-completion/completions/tkn 
kubectl completion bash > /usr/share/bash-completion/completions/kubectl
cat ${NVM_DIR}/bash_completion > /usr/share/bash-completion/completions/nvm
EOF

## Add sdkman's init script launcher to the end of ${PROFILE_EXT} since we are not adding it on sdkman install
## NOTE: all modifications to ${PROFILE_EXT} must happen BEFORE this step in order for sdkman to function correctly
RUN echo 'export SDKMAN_DIR="/home/tooling/.sdkman"' >> ${PROFILE_EXT}
RUN echo '[[ -s "$SDKMAN_DIR/bin/sdkman-init.sh" ]] && source "$SDKMAN_DIR/bin/sdkman-init.sh"' >> ${PROFILE_EXT}


# Create symbolic links from /home/tooling/ -> /home/user/
RUN stow . -t /home/user/ -d /home/tooling/ --no-folding

# Set permissions on /etc/passwd, /etc/group, /etc/pki and /home to allow arbitrary users to write
RUN chgrp -R 0 /home && chmod -R g=u /etc/passwd /etc/group /home /etc/pki

# cleanup dnf cache
RUN dnf -y clean all --enablerepo='*'

COPY --chown=0:0 entrypoint.sh /

USER 10001

ENV HOME=/home/user
