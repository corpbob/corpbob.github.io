---
layout: post
title: "Installing Keycloak on OKD 3.11"
author: "Bobby Corpus"
categories: journal
tags: [documentation,sample]
#image: 2019-08-21-introduction-to-knative/ExampleModel.png-cards2.png
---

# Assumptions

- You have OKD 3.11 up and running
- Single master setup

# Generate certificates for Keycloak to enable https

```
CA_CN="Local Keycloak Signer"
OPENSSL_CNF=/etc/pki/tls/openssl.cnf
openssl genrsa -out ca.key 4096

# Generate the root ca
openssl req -x509 \
  -new -nodes \
  -key ca.key \
  -sha256 \
  -days 1024 \
  -out ca.crt \
  -subj /CN="${CA_CN}" \
  -reqexts SAN \
  -extensions SAN \
  -config <(cat ${OPENSSL_CNF} \
      <(printf '[SAN]\nbasicConstraints=critical, CA:TRUE\nkeyUsage=keyCertSign, cRLSign, digitalSignature'))


DOMAIN=<your domain here>

# Generate domain key
openssl genrsa -out $DOMAIN-key.pem 2048

# Generate the certificate signing request for the domain:
openssl req -new -sha256 \
    -key $DOMAIN-key.pem \
    -subj "/O=Local {prod}/CN=${DOMAIN}" \
    -reqexts SAN \
    -config <(cat ${OPENSSL_CNF} \
        <(printf "\n[SAN]\nsubjectAltName=DNS:${DOMAIN}, DNS:vm$i\nbasicConstraints=critical, CA:FALSE\nkeyUsage=digitalSignature, keyEncipherment, keyAgreement, dataEncipherment\nextendedKeyUsage=serverAuth, clientAuth")) \
    -out $DOMAIN.csr

#  Generate the domain certificate:
openssl x509 \
    -req \
    -sha256 \
    -extfile <(printf "subjectAltName=DNS:${DOMAIN}, DNS:vm$i\nbasicConstraints=critical, CA:FALSE\nkeyUsage=digitalSignature, keyEncipherment, keyAgreement, dataEncipherment\nextendedKeyUsage=serverAuth, clientAuth") \
    -days 365 \
    -in $DOMAIN.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial -out $DOMAIN.crt

```

- Copy the certificates to a directory, say ```certs```

```
cp $DOMAIN.crt certs/tls.crt
cp $DOMAIN-key.pem certs/tls.key
```

- Create a config map or secret out of the certificates

```
oc create configmap certificates --from-file=certs
```

- Copy the file ```$DOMAIN.crt``` to the master nodes directory ```/etc/origin/master```

# Keycloak Installation and Configuration
- Deploy the keycloak image

```
oc new-project keycloak
oc new-app jboss/keycloak
```

- Mount the certificates

```
oc set volume dc keycloak --add --type configmap --configmap-name certificates --mount-path /etc/x509/https
```

- Set the username and password for admin in the deployment config

```
oc set env dc/keycloak KEYCLOAK_USER=admin
oc set env dc/keycloak KEYCLOAK_PASSWORD=<password here>
```

*Note* Best practice is to create a secret for the credentials and reference the username and password in the above

- This will redeploy keycloak. It will import the certificates into it's keystore and will set the admin user credentials

- Give this a route

```
oc create route passthrough keycloak --hostname=$DOMAIN --service=keycloak --port=8443
```

- Login to keycloak and create a realm called ```openshift```
- Create a user, say ```test-user``` with password ```password```. Make sure that ```Temporary Password``` is set to Off.
- Create a client called ```openshift``` and set the client protocol to ```openid-connect``` and access type ```confidential```
- Copy the client secret. This will be needed in configuring openshift 3.11

# OKD 3.11 configuration

- Edit the file /etc/origin/master-config.xml (back up always).
- Find the identityProviders section and add the following:
*Important* Be sure to substitute the correct value of $DOMAIN

```
  - name: rh_sso
    challenge: false
    login: true
    mappingInfo: add
    provider:
      apiVersion: v1
      kind: OpenIDIdentityProvider
      clientID: openshift
      clientSecret: g8d7f50b-d781-4c4c-baa6-adbe4b76a280
      ca: $DOMAIN.crt
      urls:
        authorize: $DOMAIN/auth/realms/openshift/protocol/openid-connect/auth
        token: $DOMAIN/auth/realms/openshift/protocol/openid-connect/token
        userInfo: $DOMAIN/auth/realms/openshift/protocol/openid-connect/userinfo
        logoutURL: $DOMAIN/auth/realms/openshift/protocol/openid-connect/logout
      claims:
        id:
        - sub
        preferredUsername:
        - preferred_username
        name:
        - name
        email:
        - email
```
*Note* Take note where we specify the crt file in the above configuration.

- restart the masters

```
sudo /usr/local/bin/master-restart api && sudo /usr/local/bin/master-restart controllers
```

# Testing

- Logout of OKD and login again. You will see two ways to login, one using htpasswd and the other using rh_sso
- Use rh_sso and login using the user you created previously
- You will then be redirected to the keycloak login page. Login using ```testuser``` and password ```password```
- You should be able to login successfully in OKD.

# Troubleshootting
- Initially I have failed authentication. To troubleshoot it, I tailed the logs of the pod master-api in ```kube-system```. There i found out that the certificate was for the wrong domain:

```
Error getting access token: Post <token url>: x509: certificate is valid for <some domain>, not <your keycloak domain>
``` 
- From there you will know what to do.
