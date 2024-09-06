# syntax=docker/dockerfile:1.5
ARG IGNITION_VERSION=${IGNITION_VERSION}
FROM inductiveautomation/ignition:${IGNITION_VERSION} as prep

# Temporarily become root for system-level updates (required for 8.1.26+)
USER root

# Install some prerequisite packages
RUN apt-get update && apt-get install -y wget ca-certificates jq zip unzip sqlite3

ARG SUPPLEMENTAL_AWSINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.22/AWS-Injector-signed.modl"
ARG SUPPLEMENTAL_AWSINJECTOR_DOWNLOAD_SHA256="110b4794c6e046f065635c04fb28504724feb7e6eca8f80f1b493212eb9bc09d"
ARG SUPPLEMENTAL_AZUREIOTINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.22/Azure-Injector-signed.modl"
ARG SUPPLEMENTAL_AZUREIOTINJECTOR_DOWNLOAD_SHA256="1eeb29aac155738433eb570c1a622f019fa9053d2dc88fa5a18586f1d5516ef3"
ARG SUPPLEMENTAL_GCPINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.22/Google-Cloud-Injector-signed.modl"
ARG SUPPLEMENTAL_GCPINJECTOR_DOWNLOAD_SHA256="16817e8ce58c27f53cb2aa511add1d91b3df2a9f4fd2608c9732acee0f267026"
ARG SUPPLEMENTAL_MQTTTRANSMISSION_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.22/MQTT-Transmission-signed.modl"
ARG SUPPLEMENTAL_MQTTTRANSMISSION_DOWNLOAD_SHA256="0c223f44e687ca03a2eb64854895bca93f4535f8bdc709995d309c8df53f1aa3"
ARG SUPPLEMENTAL_MQTTTRANSMISSIONNIGHTLY_DOWNLOAD_URL="https://ignition-modules-nightly.s3.amazonaws.com/Ignition8/MQTT-Transmission-signed.modl"
ARG SUPPLEMENTAL_MQTTTRANSMISSIONNIGHTLY_DOWNLOAD_SHA256="notused"
ARG SUPPLEMENTAL_MQTTENGINE_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.22/MQTT-Engine-signed.modl"
ARG SUPPLEMENTAL_MQTTENGINE_DOWNLOAD_SHA256="affc55e4e3b011eb44cdf63f864307f03b6fb28a7c8c53e886394d517caa21e5"
ARG SUPPLEMENTAL_MQTTENGINENIGHTLY_DOWNLOAD_URL="https://ignition-modules-nightly.s3.amazonaws.com/Ignition8/MQTT-Engine-signed.modl"
ARG SUPPLEMENTAL_MQTTENGINENIGHTLY_DOWNLOAD_SHA256="notused"
ARG SUPPLEMENTAL_MQTTDISTRIBUTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.22/MQTT-Distributor-signed.modl"
ARG SUPPLEMENTAL_MQTTDISTRIBUTOR_DOWNLOAD_SHA256="4485d06f1a44745de80dd9facb63d2286784f643192ca25b19a3a77eb61e71e8"
ARG SUPPLEMENTAL_MQTTDISTRIBUTORNIGHTLY_DOWNLOAD_URL="https://ignition-modules-nightly.s3.amazonaws.com/Ignition8/MQTT-Distributor-signed.modl"
ARG SUPPLEMENTAL_MQTTDISTRIBUTORNIGHTLY_DOWNLOAD_SHA256="notused"
ARG SUPPLEMENTAL_MODULES


# Set working directory for this prep image and ensure that exits from sub-shells bubble up and report an error
WORKDIR /root
SHELL [ "/usr/bin/env", "-S", "bash", "-euo", "pipefail", "-O", "inherit_errexit", "-c" ]

# Retrieve all targeted modules and verify their integrity
COPY --chmod=0755 retrieve-modules.sh .
RUN ./retrieve-modules.sh \
    -m "${SUPPLEMENTAL_MODULES:-}"

# Set CERTIFICATES/EULAS acceptance in gateway backup config db
COPY base.gwbk .
COPY --chmod=0755 register-module.sh register-password.sh ./
ARG GATEWAY_ADMIN_USERNAME="admin"
RUN --mount=type=secret,id=gateway-admin-password \
    unzip -q base.gwbk db_backup_sqlite.idb && \
    shopt -s nullglob; \
    for module in *.modl; do \
    ./register-module.sh \
      -f "${module}" \
      -d db_backup_sqlite.idb; \
    done; \
    shopt -u nullglob && \
    ./register-password.sh \
      -u "${GATEWAY_ADMIN_USERNAME}" \
      -f /run/secrets/gateway-admin-password \
      -d db_backup_sqlite.idb && \
    zip -q -f base.gwbk db_backup_sqlite.idb || \
    if [[ ${ZIP_EXIT_CODE:=$?} == 12 ]]; then \
    echo "No changes to internal database needed during module registration."; \
    else \
    echo "Unknown error (${ZIP_EXIT_CODE}) encountered during re-packaging of config db, exiting." && \
    exit ${ZIP_EXIT_CODE}; \
    fi

# Final Image
FROM inductiveautomation/ignition:${IGNITION_VERSION} as final

# Temporarily become root for system-level updates (required for 8.1.26+)
USER root

# Add supplemental packages, such as git if needed/desired
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
    git && \
    rm -rf /var/lib/apt/lists/*

# Embed modules and base gwbk from prep image as well as entrypoint shim
COPY --from=prep --chown=ignition:ignition /root/*.modl ${IGNITION_INSTALL_LOCATION}/user-lib/modules/
COPY --from=prep --chown=ignition:ignition /root/base.gwbk ${IGNITION_INSTALL_LOCATION}/base.gwbk
COPY --chmod=0755 --chown=root:root docker-entrypoint-shim.sh /usr/local/bin/

# Return to ignition user
USER ignition

# Supplement other default environment variables
ENV ACCEPT_IGNITION_EULA=Y \
    IGNITION_EDITION=standard \
    GATEWAY_MODULES_ENABLED=all

# Target the entrypoint shim for any custom logic prior to gateway launch
ENTRYPOINT [ "docker-entrypoint-shim.sh" ]
