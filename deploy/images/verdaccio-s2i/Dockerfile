FROM quay.io/erikerlandson/verdaccio-ubi:s2i

# s2i scripts assemble, run
LABEL io.openshift.s2i.scripts-url=image:///opt/verdaccio/s2i

# s2i source loads here
LABEL io.openshift.s2i.destination=image:///opt/verdaccio/src

COPY ./s2i/ /opt/verdaccio/s2i/bin/
COPY ./config.yaml /opt/verdaccio/s2i/

USER root

RUN chown -R 9999:0 /opt/verdaccio/s2i \
 && chmod -R g+rwX /opt/verdaccio/s2i \
 && mkdir /opt/verdaccio/src \
 && chown -R 9999:0 /opt/verdaccio/src \
 && chmod g+rwX /opt/verdaccio/src

# emulate anonymous uid
USER 99999
