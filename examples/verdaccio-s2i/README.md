# Deploying verdaccio with S2I

This directory contains a tutorial example deployment of verdaccio using OpenShift
[source-to-image](https://github.com/openshift/source-to-image)
(S2I).

### TLS certificate prerequisite
The tutorial assumes you have created an OpenShift secret named `verdaccio-cert` that contains
TLS certs that enable OpenShift to expose the verdaccio service in `https` mode.
You can create this secret yourself, or follow the instructions in the `tls-certs` subdirectory of this example.
If you do create this secret on your own, be sure to install the `cert-manager` and `cert-utils` operators from
the OpenShift Operator Catalog
(as described in the `tls-certs` subdirectory).

### verdaccio S2I conventions
The verdaccio S2I builder image expects a context directory that may contain two files:
* `package.json` - an npm package file
* `config.yaml` - a verdaccio config file

Both of these files are optional.

If `package.json` is present, then the S2I build process will run `npm install` using this package
file and cause verdaccio to cache the installed packages. This cache will be preserved in the ouptput
image; in other words, it will build a verdaccio server that has these packages (and their dependencies)
pre-cached.  If `package.json` is not present, then the resulting image will have no pre-cached dependencies.

If `config.yaml` is present then this file will be installed and the verdaccio server will use it as its
config file. Otherwise, it will install a
[default](https://github.com/erikerlandson/verdaccio-openshift/blob/main/deploy/images/verdaccio-s2i/config.yaml)
configuration file.

The `config` sub-directory of this tutorial contains examples of each of these files.
In the section that follows you can see how to apply S2I to this directory to build a verdaccio server.

### Using an OpenShift Build Configuration

One useful way to deploy a verdaccio server using S2I is by installing a BuildConfig onto your
OpenShift project that is configured to build your server with the `verdaccio-s2i` image.

Here is an example BuildConfig definition that will build a verdaccio server image from
the `config` subdirectory of this tutorial:

```yaml
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
  name: verdaccio-s2i-example
spec:
  output:
    to:
      kind: ImageStreamTag
      name: 'verdaccio-s2i-example:latest'
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: DockerImage
        name: 'quay.io/verdaccio-openshift/verdaccio-s2i:0.1.1'
  source:
    type: Git
    git:
      uri: 'https://github.com/erikerlandson/verdaccio-openshift.git'
      ref: main
    contextDir: examples/verdaccio-s2i/config
  triggers:
    - type: ConfigChange
  runPolicy: Serial
```

### An example S2I verdaccio deployment

The preceding BuildConfig will build an image that you an run as a verdaccio server.
However, a "complete" deployment will usually require some other objects.

You can create a fully functional verdaccio deployment in OpenShift by installing the objects in the
manifest file `verdaccio-s2i-example.yaml` in this tutorial directory.
Note that you will probably want to edit this YMAL to replace `verdaccio.manyangled.dev` with the
hostname that you use (refer to the instructions in the tls-cert directory).

```sh
$ cd /path/to/verdaccio-openshift/examples/verdaccio-s2i
$ oc apply -f verdaccio-s2i-example.yaml
imagestream.image.openshift.io/verdaccio-s2i-example created
buildconfig.build.openshift.io/verdaccio-s2i-example created
service/verdaccio-s2i-example created
route.route.openshift.io/verdaccio-s2i-example created
deploymentconfig.apps.openshift.io/verdaccio-s2i-example created
```

The manifest above installs the following objects to your OpenShift project:
* An ImageStream to hold your verdaccio server image
* A BuildConfig to build your server with S2I (see section above)
* A Service to expose your verdaccio server API
* A Route to allow you to access your server from outside of OpenShift
* A DeploymentConfig to run your verdaccio image and specify mounts, environment vars, and other
forms of depenency injections to your server image.

### Using Your verdaccio Deployment

You can test your verdaccio deployment with `npm`.
You'll want to replace `verdaccio.manyangled.dev` with the hostname you used.

```sh
$ npm install react --registry https://verdaccio.manyangled.dev

# ... you may see some WARN messages here ...

+ react@17.0.1
added 4 packages from 3 contributors and audited 4 packages in 1.361s
found 0 vulnerabilities
```

If your TLS certs were generated properly, the above npm test should work.
However, if npm complains about certs that are self-signed or other problems,
it may work to disable strict SSL:

```sh
# disable strict SSL if your TLS certs are self-signed or cause npm to complain
$ export npm_config_strict_ssl=false
```

### deleting the deployment

To delete the verdaccio deployment objects run the following on your command line:
```sh
oc delete deploymentconfig/verdaccio-s2i-example \
          route/verdaccio-s2i-example \
          service/verdaccio-s2i-example \
          buildconfig/verdaccio-s2i-example \
          imagestream/verdaccio-s2i-example
```

If you installed the TLS certificates using the instructions in the
[cert-manager](https://github.com/erikerlandson/verdaccio-openshift/tree/main/examples/cert-manager)
example, you can delete them like this:
```sh
oc delete certificate/verdaccio-certificate \
          issuer/gcp-dns01-staging \
          issuer/gcp-dns01-production \
          secret/verdaccio-cert
```
