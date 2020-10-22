# verdaccio openshift images

### get verdaccio-openshift images

Images built from this repository are currently hosted at the verdaccio-openshift organization on quay.io
* https://quay.io/organization/verdaccio-openshift

### Configuring verdaccio

The verdaccio server is
[configured](https://verdaccio.org/docs/en/configuration)
via a `config.yaml` file.
You can customize your verdaccio `config.yaml` file in multiple ways.
* Create a custom verdaccio server from source-to-image, using the `verdaccio-s2i` builder image
* Run the `verdaccio-ubi` image with a config file mounted to your container (for example from a ConfigMap),
and set the `VERDACCIO_CONFIG` to the corresponding path.
* Build your own customized image with a config.yaml preloaded.

The default verdaccio-openshift startup command is `/opt/verdaccio/bin/run-verdaccio`.
This script is aware of the `VERDACCIO_CONFIG` environment variable,
and will start up verdaccio with the config.yaml path in this variable if it is set.

When building verdaccio images via source-to-image using `verdaccio-s2i`,
a default `config.yaml` will be installed if none is supplied in your s2i source.
This default config will not have `https` capability configured.
To build an image with proper TLS support, the recommended approach is to
include your own config.yaml file in the s2i build, that configures TLS cert paths
corresponding to where you plan to mount them to your image.

### TLS Cert Provisioning

Many tools in the nodejs ecosystem require a secure `https` npm registry URL by default.
The verdaccio-openshift images run verdaccio using `http` by default,
and can be configured for `https` by setting the environment variable `VERDACCIO_PROTOCOL=https`,
for example via the OpenShift or K8s downward API.

In order to support `https`, verdaccio needs TLS certifications.
These are configured in the verdaccio `config.yaml` file, as in this example config fragment:
```yaml
https:
  key: /opt/verdaccio/cert/tls.key
  cert: /opt/verdaccio/cert/tls.crt
```

The TLS key and cert filenames in this example are taken from files created by mounting a Kubernetes or OpenShift secret to the container, using directory `/opt/verdaccio/cert` as the mount-point.
The fields `tls.key` and `tls.cert` are standard names populated by the `cert-manager` operator, which can be installed from the OpenShift operator catalog.
You can also populate a secret with a TLS key and cert manually,
as long as your verdaccio config.yaml settings correspond to your secret field names.

verdaccio can run `https` using "self-signed" TLS certificates.
However, tools like `npm` will not by default connect to an npm registry using self-signed certs.
If your TLS certs are self-signed (or are self-signed somewhere in the signing chain),
then you can disable strict TLS using:
```sh
$ npm config set strict-ssl false --global
```
or alternatively using environment vars:
```sh
$ export npm_config_strict_ssl=false
```

Another safer alternative is to set npm with the TLS certs you are using:
```sh
$ npm config set cafile /path/to/your/cert.pem --global
```

For more information on working with self-signed certificates in the nodejs ecosystem,
refer to nodejs tooling documentation or
[community articles](https://medium.com/@jonatascastro12/understanding-self-signed-certificate-in-chain-issues-on-node-js-npm-git-and-other-applications-ad88547e7028).
