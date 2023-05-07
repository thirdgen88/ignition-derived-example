# syntax=docker/dockerfile:1.5
ARG IGNITION_VERSION=${IGNITION_VERSION}
FROM inductiveautomation/ignition:${IGNITION_VERSION} as prep

# Temporarily become root for system-level updates (required for 8.1.26+)
USER root

# Install some prerequisite packages
RUN apt-get update && apt-get install -y wget ca-certificates jq zip unzip sqlite3

ARG SUPPLEMENTAL_AZUREIOTINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.16/Azure-Injector-signed.modl"
ARG SUPPLEMENTAL_AZUREIOTINJECTOR_DOWNLOAD_SHA256="acca89ac89fce21ac76dca91f930a03753fbbe1654f4335bee7154e709aa6f66"
ARG SUPPLEMENTAL_AWSINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.16/AWS-Injector-signed.modl"
ARG SUPPLEMENTAL_AWSINJECTOR_DOWNLOAD_SHA256="2298abf3d92be5c8c63331e66ffd4f1aeeca8b6fc1043d30d133055aee8cdf86"
ARG SUPPLEMENTAL_GOOGLECLOUDINJECTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.16/Google-Cloud-Injector-signed.modl"
ARG SUPPLEMENTAL_GOOGLECLOUDINJECTOR_DOWNLOAD_SHA256="58df78838e14315349cf2d6f72259da483898da57792b384e16918f2d392795e"
ARG SUPPLEMENTAL_MQTTTRANSMISSION_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.16/MQTT-Transmission-signed.modl"
ARG SUPPLEMENTAL_MQTTTRANSMISSION_DOWNLOAD_SHA256="59d9334196284ee449816269a60ed74f918f2ec093e31086271a454f04167208"
ARG SUPPLEMENTAL_MQTTTRANSMISSIONNIGHTLY_DOWNLOAD_URL="https://ignition-modules-nightly.s3.amazonaws.com/Ignition8/MQTT-Transmission-signed.modl"
ARG SUPPLEMENTAL_MQTTTRANSMISSIONNIGHTLY_DOWNLOAD_SHA256="notused"
ARG SUPPLEMENTAL_MQTTENGINE_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.16/MQTT-Engine-signed.modl"
ARG SUPPLEMENTAL_MQTTENGINE_DOWNLOAD_SHA256="b16cdab50c624d1d7c8e62770ef416447bc6340ac98acbd582c70e924ca53eee"
ARG SUPPLEMENTAL_MQTTENGINENIGHTLY_DOWNLOAD_URL="https://ignition-modules-nightly.s3.amazonaws.com/Ignition8/MQTT-Engine-signed.modl"
ARG SUPPLEMENTAL_MQTTENGINENIGHTLY_DOWNLOAD_SHA256="notused"
ARG SUPPLEMENTAL_MQTTDISTRIBUTOR_DOWNLOAD_URL="https://files.inductiveautomation.com/third-party/cirrus-link/4.0.16/MQTT-Distributor-signed.modl"
ARG SUPPLEMENTAL_MQTTDISTRIBUTOR_DOWNLOAD_SHA256="899e170b8ceb81800ab57b0e4c729db0dddfe7bef6199a30f127efe31ef6d8e2"
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
