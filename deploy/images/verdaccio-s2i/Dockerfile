FROM quay.io/erikerlandson/verdaccio-ubi:s2i

USER root

RUN microdnf install tar \
 && microdnf clean all

# s2i scripts assemble, run
LABEL io.openshift.s2i.scripts-url=image:///opt/verdaccio/s2i/bin

# s2i source loads here
LABEL io.openshift.s2i.destination=/opt/verdaccio

COPY ./s2i/ /opt/verdaccio/s2i/bin/
COPY ./config.yaml /opt/verdaccio/s2i/

RUN chown -R 9999:0 /opt/verdaccio/s2i \
 && chmod -R g+rwX /opt/verdaccio/s2i \
 && mkdir /opt/verdaccio/src \
 && chown -R 9999:0 /opt/verdaccio/src \
 && chmod g+rwX /opt/verdaccio/src

# emulate anonymous uid
USER 99999
