FROM registry.access.redhat.com/ubi8/ubi-minimal:8.1

RUN microdnf install nodejs npm \
 && microdnf update \
 && microdnf clean all

WORKDIR /opt/verdaccio

# defines the packages to run verdaccio and any plugins
COPY package.json ./

# script that starts up verdaccio server
COPY run-verdaccio ./bin/

# Install verdaccio and plugins.
# make sure resulting installs are readable by gid 0
RUN npm install \
 && chown -R 9999:0 /opt/verdaccio \
 && chmod a+rx /opt/verdaccio/bin/* \
 && chmod -R g+rwX /opt/verdaccio

ENV PATH="/opt/verdaccio/node_modules/verdaccio/bin:$PATH" \
    VERDACCIO_PROTOCOL=http \
    VERDACCIO_PORT=4873

# emulate an anonymous uid, which is better aligned with OpenShift environment
USER 99999

# default CMD is to run with no config.yaml (mostly default settings)
CMD /opt/verdaccio/bin/run-verdaccio
