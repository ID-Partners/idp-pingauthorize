# syntax=docker/dockerfile:1
# PingAuthorize 11 with the AuthZEN Adapter installed, deciding with the example policy.
FROM pingidentity/pingauthorize:11.1.0.0-latest@sha256:7360ae0b5a869173322e1ce2e03e641da7f8525b9be41ad9bc3129d7ac4158f4

ENV PING_IDENTITY_ACCEPT_EULA=YES
ENV MAX_HEAP_SIZE=1g

# The released bundle for PingAuthorize 11, checked against the SHA256SUMS published with it.
USER root
ADD --checksum=sha256:50c9d7026ac8d4aa38fe1179fc2ecef3e1784f63539958927b41fab7454f3656 \
    https://github.com/ID-Partners/idp-pingauthorize/releases/download/authzen-adapter-v2.0.0/authzen-adapter-2.0.0-paz11.zip \
    /tmp/authzen-adapter.zip
# Installed into the image, not delivered through the server profile: on 11.1 the profile route
# stops setup with "Server Already Configured".
RUN /opt/server/bin/manage-extension --install /tmp/authzen-adapter.zip --no-prompt --ignoreWarnings \
    && rm -f /tmp/authzen-adapter.zip \
    && chown -R 9031:0 /opt/server
USER 9031:0

COPY --chown=9031:0 pd.profile /opt/in/pd.profile
COPY --chown=9031:0 policy/authzen-example.deploymentpackage /opt/in/authzen-example.deploymentpackage

CMD ["start-server"]
