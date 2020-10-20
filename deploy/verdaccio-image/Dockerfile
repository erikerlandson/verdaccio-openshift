FROM verdaccio/verdaccio

USER root

# installing the verdaccio container packages.json makes
# the offline storage plugin install properly
RUN npm install && npm install verdaccio-offline-storage

# set up a pre-populated package cache from tarball
COPY vcache.tgz /tmp/
RUN mkdir -p /verdaccio/cache \
 && chown verdaccio /verdaccio/cache \
 && cd /verdaccio/cache \
 && tar xzf /tmp/vcache.tgz \
 && rm /tmp/vcache.tgz \
 && chown -R verdaccio:nogroup /verdaccio/cache/data

# use this verdaccio config file
# includes config to use offline storage
COPY config.yaml /verdaccio/conf/

# these self-signed certs don't work with npm install
# although the verdaccio server will start up with them
#COPY verdaccio-key.pem /verdaccio/conf/
#COPY verdaccio-cert.pem /verdaccio/conf/
#RUN chown verdaccio /verdaccio/conf/* \
# && chmod 664 /verdaccio/conf/*.pem
#ENV VERDACCIO_PROTOCOL='https' 

USER verdaccio
