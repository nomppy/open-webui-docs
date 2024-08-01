---
sidebar_position: 14
title: Setting up with custom CA store
---

If you get an `[SSL: CERTIFICATE_VERIFY_FAILED]` error when trying to run OI, most likely the issue is that you are on a network which intercepts HTTPS traffic (e.g. a corporate network).

To fix this, you will need to add the new cert into OI's truststore.

**For pre-built Docker image**:

1. Mount the certificiate store from your host machine into the container by passing `--volume=/etc/ssl/certs/ca-certificiate.crt:/etc/ssl/certs/ca-certificiates.crt:ro` as a command-line option to `docker run`
2. Force python to use the system truststore by setting `REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt` (see the [Docker documentation](https://docs.docker.com/reference/cli/docker/container/run/#env))

In a compose file, it might look like this

```
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    volumes:
      - /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.crt:ro
    environment:
      - REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
      - SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
```

You can also mount a different CA than the system one, like in this trimmed-down example `compose.yaml` from [@KizzyCode](https://github.com/open-webui/open-webui/issues/1398#issuecomment-2258463210):

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    volumes:
      - /var/containers/openwebui:/app/backend/data:rw
      - /etc/containers/openwebui/compusrv.crt:/etc/ssl/certs/ca-certificates.crt:ro
    environment:
      - REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
```

The `ro` flag mounts the CA store as read-only and prevents accidental changes to your host CA store

**For local development**:

If you want to set up a local [dev environment](https://docs.openwebui.com/tutorial-development/), you can also include the certificates in the build process by
by modifying the `Dockerfile`.
Since the build happens in [multiple stages](https://docs.docker.com/build/building/multi-stage/), you have to add the cert into both

1. Frontend (`build` stage):

```dockerfile
# copy your cert file to the build context
COPY package.json package-lock.json <YourRootCert>.crt ./       
# instruct node to use it
ENV NODE_EXTRA_CA_CERTS=/app/<YourRootCert>.crt                 
RUN npm ci
```

2. Backend (`base` stage)

```dockerfile
COPY <CorporateSSL.crt> /usr/local/share/ca-certificates/       
RUN update-ca-certificates
ENV PIP_CERT=/etc/ssl/certs/ca-certificates.crt \               
    REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \    
    SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
```
