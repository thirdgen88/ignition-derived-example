# syntax=docker/dockerfile:1.14
ARG IGNITION_VERSION
FROM inductiveautomation/ignition:${IGNITION_VERSION:-latest} AS prep

# Temporarily become root for system-level updates (required for 8.1.26+)
USER root

# Install some prerequisite packages
RUN apt-get update && apt-get install -y wget ca-certificates jq zip unzip sqlite3

ARG SUPPLEMENTAL_AWSINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.26/AWS-Injector-signed.modl"
ARG SUPPLEMENTAL_AWSINJECTOR_DOWNLOAD_SHA256="f3f3be84c3a4972a2c711382fe0780fcfc3acf152207b90ab7ab95fccff1a623"
ARG SUPPLEMENTAL_AZUREIOTINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.26/Azure-Injector-signed.modl"
ARG SUPPLEMENTAL_AZUREIOTINJECTOR_DOWNLOAD_SHA256="dfbbab663aba90a29f382efc216341a953f12ec68e23882ec0d13c8c8e9fb054"
ARG SUPPLEMENTAL_GCPINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.26/Google-Cloud-Injector-signed.modl"
ARG SUPPLEMENTAL_GCPINJECTOR_DOWNLOAD_SHA256="43071699bc309395dadd2c124ad0e4172ee6c79b9cc7e93c3f88224f7dc97dac"
ARG SUPPLEMENTAL_MQTTTRANSMISSION_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.26/MQTT-Transmission-signed.modl"
ARG SUPPLEMENTAL_MQTTTRANSMISSION_DOWNLOAD_SHA256="fdf8ae733ec5a82218977e7b20bc2afda0423adfd1a127ae8c21ce27bf5bc6c7"
ARG SUPPLEMENTAL_MQTTTRANSMISSIONNIGHTLY_DOWNLOAD_URL="https://ignition-modules-nightly.s3.amazonaws.com/Ignition8/MQTT-Transmission-signed.modl"
ARG SUPPLEMENTAL_MQTTTRANSMISSIONNIGHTLY_DOWNLOAD_SHA256="notused"
ARG SUPPLEMENTAL_MQTTENGINE_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.26/MQTT-Engine-signed.modl"
ARG SUPPLEMENTAL_MQTTENGINE_DOWNLOAD_SHA256="f6ef3c575b9b6aa04deebeb040b27b734629908c25369fb4ff1e26a00d37f098"
ARG SUPPLEMENTAL_MQTTENGINENIGHTLY_DOWNLOAD_URL="https://ignition-modules-nightly.s3.amazonaws.com/Ignition8/MQTT-Engine-signed.modl"
ARG SUPPLEMENTAL_MQTTENGINENIGHTLY_DOWNLOAD_SHA256="notused"
ARG SUPPLEMENTAL_MQTTDISTRIBUTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.26/MQTT-Distributor-signed.modl"
ARG SUPPLEMENTAL_MQTTDISTRIBUTOR_DOWNLOAD_SHA256="7d1e3ed9700228b8d41b3b7730bc7e058f0be4ddbaeb6d635b4c196355c44e34"
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
FROM inductiveautomation/ignition:${IGNITION_VERSION:-latest} AS final

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
