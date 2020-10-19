FROM registry.access.redhat.com/ubi8/ubi-minimal:8.1

RUN microdnf install nodejs npm \
 && microdnf update \
 && microdnf clean all \
 && chmod g+w /etc/passwd

WORKDIR /opt/verdaccio
COPY verdaccio-package.json ./package.json
COPY verdaccio-config.yaml ./

# easy way to get npm to cooperate about using /tmp during install
RUN adduser --uid 9999 --home-dir /tmp --gid 0 -s /sbin/nologin tvu \
 && chown -R tvu /opt/verdaccio

USER tvu

RUN npm install \
 && chmod -R g+rwX /opt/verdaccio

ENV PATH="/opt/verdaccio/node_modules/verdaccio/bin:$PATH"

# spin up verdaccio as a transient service and run npm to pre-populate its cache
RUN (verdaccio &) \
 && sleep 5 \
 && cd /tmp \
 && npm install vega vega-lite iconv-lite@0.4 --registry http://localhost:4873 \
 && rm -rf /tmp/node_modules \
 && chmod -R g+rwX /opt/verdaccio/verdaccio

# emulate anonymous uid
USER 99999

ENV VERDACCIO_PROTOCOL=http VERDACCIO_PORT=4873

CMD verdaccio --config verdaccio-config.yaml --listen $VERDACCIO_PROTOCOL://0.0.0.0:$VERDACCIO_PORT
