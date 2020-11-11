# TLS With the cert-manager Operator

The verdaccio-openshift images can be configured to mount TLS certificates from any secret.
However, the OpenShift
[OperatorHub](https://docs.openshift.com/container-platform/4.5/operators/understanding/olm-understanding-operatorhub.html)
provides tools that can make TLS certificate generation and
management easier.
One popular example is the
[cert-manager](https://cert-manager.io/docs/)
operator.
You can subscribe to this operator (as cluster admin) from OpenShift OperatorHub and
deploy a cert-manager instance into your OpenShift project (as a developer).

Once you have deployed cert-manager into your OpenShift project,
generating TLS certificates for verdaccio images is easy;
it requires installing only two objects into your project, as described below.

First, you need to define a certificate Issuer.
cert-manager supports a
[variety](https://cert-manager.io/docs/configuration/#supported-issuer-types)
of Issuer types.
In this example we'll be creating a CA Issuer, which looks like this:

```yaml
apiVersion: cert-manager.io/v1alpha3
kind: Issuer
metadata:
  name: ca-issuer
spec:
  # declare a CA certificate issuer
  # cert-manager supports several kinds of issuer
  ca:
    # using a secret already created by cert-manager operator
    # this could be some other secret containing a CA bundle
    secretName: cert-manager-webhook-ca
```

As you can see in the object above, this issuer is configured with a TLS CA stored on a secret.
In this example, I am using a CA secret that cert-manager creates automatically when it is deployed.
You can see what this secret looks like in the file `cert-manager-webhook-ca.yaml` in this directory.
If you wish, you can populate your own secret with a different CA bundle.

The Issuer above can be installed into your project from the yaml files in this directory:
```sh
$ oc apply -f ca-issuer.yaml
```

Next you need to create a Certificate object, to tell cert-manager you want it to generate a TLS cert.
A Certificate for generating verdaccio TLS certs might look like this:
```yaml
apiVersion: cert-manager.io/v1alpha3
kind: Certificate
metadata:
  name: verdaccio-certificate
spec:
  # secret containing certs is created by cert-manager
  secretName: verdaccio-cert
  commonName: verdaccio-cert
  issuerRef:
    name: ca-issuer
    kind: Issuer
    group: cert-manager.io
```

Notice that we specifically referenced the CA Issuer object ('ca-issuer') that we defined above.
We also configured a secret name to host our generated certificates.
The 'verdaccio-cert' secret will be created automatically by cert-manager once it detects
your Certificate.

You can install this Certificate into your project from the files in this directory with:
```sh
$ oc apply -f verdaccio-certificate.yaml
```

Once you install this Certificate, the new TLS CA secret 'verdaccio-cert' will be created,
and you can mount this secret to your verdaccio image and configure verdaccio
to use the TLS certs to support a secure `https` protocol.

One final note:
The TLS certificates that are created by the CA Issure we configured above have "self-signed"
certificates in their chain.
That is no problem for verdaccio, but tools such as `npm` will detect self-signed TLS certs and
error out, unless they are
[configured](https://medium.com/@jonatascastro12/understanding-self-signed-certificate-in-chain-issues-on-node-js-npm-git-and-other-applications-ad88547e7028)
to either allow non-strict SSL, or configured with the cert file.
