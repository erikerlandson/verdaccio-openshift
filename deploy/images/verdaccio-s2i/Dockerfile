FROM quay.io/verdaccio-openshift/verdaccio-ubi:0.1.0

USER root

# UBI minimal does not come with tar installed, and s2i needs it
RUN microdnf install tar \
 && microdnf clean all

# s2i scripts assemble, run
LABEL io.openshift.s2i.scripts-url=image:///opt/verdaccio/s2i/bin

# s2i source loads here
# The s2i process creates the 'src' directory, so don't add it here
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
