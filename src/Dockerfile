# syntax=docker/dockerfile:1
# check=skip=SecretsUsedInArgOrEnv
FROM alpine:latest

ENV ALLOWED_IPS="AllowUsers tunnel" \
    DEBUG_MODE="false" \
    PGID=9999 \
    PUID=9999 \
    SSH_HOST_KEY_DIR="/etc/ssh/ssh_host_keys" \
    SSH_PORT="2222" \
    SSH_USER="tunnel" \
    SSH_GATEWAYPORTS="no"

ARG PACKAGE_DEPENDENCIES="openssh shadow tini" \
    REPOSITORY_BUILD_VERSION="dev"

COPY --chown=root:root --chmod=755 src/rootfs /

RUN apk add --no-cache ${PACKAGE_DEPENDENCIES} && \
    \
    # Ensure /usr/bin/tini exists if installed in /sbin/tini
    ([ -e /usr/bin/tini ] || ln -s /sbin/tini /usr/bin/tini) && \
    \
    # Create unprivileged user
    serversideup-create-unprivileged-user "$SSH_USER" "${PUID}" "${PGID}" && \
    \
    # Set proper permissions
    mkdir -p /home/$SSH_USER/.ssh $SSH_HOST_KEY_DIR && \
    chown -R $SSH_USER:$SSH_USER $SSH_HOST_KEY_DIR /home/$SSH_USER/.ssh && \
    chmod 700 /home/$SSH_USER/.ssh && \
    \
    # Create run directory
    mkdir -p /run/sshd /var/empty && \
    \
    # Remove default SSH host keys
    rm -f /etc/ssh/ssh_host_*_key*

# Expose the SSH port
EXPOSE 2222

LABEL org.opencontainers.image.title="ays7/ssh-tunnels-server" \
    org.opencontainers.image.description="Hardened SSH tunnel container. Exclusively for port forwarding and secure connections into clusters." \
    org.opencontainers.image.url="https://github.com/ays7/docker-ssh" \
    org.opencontainers.image.source="https://github.com/ays7/docker-ssh" \
    org.opencontainers.image.documentation="https://github.com/ays7/docker-ssh" \
    org.opencontainers.image.vendor="ays7" \
    org.opencontainers.image.authors="Alexey (@ays7), Jay Rogers (@jaydrogers)" \
    org.opencontainers.image.version="${REPOSITORY_BUILD_VERSION}" \
    org.opencontainers.image.licenses="GPL-3.0-or-later"

HEALTHCHECK --interval=30s --timeout=30s --start-period=30s --retries=3 \
    CMD pgrep -f "sshd.*-D" > /dev/null || exit 1

ENTRYPOINT ["/usr/bin/tini", "--", "/entrypoint.sh"]
CMD ["/usr/sbin/sshd", "-D", "-e"]