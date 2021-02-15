# Enable SSL/TLS for a Kubernetes application

In this lab, we will go through the steps of enabling SSL/TLS for your cluster application.

## Pre-reqs

Before you begin with this lab, you have to complete the previous one:

* [Expose an application with an ingress](./02-deploy-ingress.md)

## Tasks

### Deploy cert-manager to your cluster

In order to enable SSL/TLS, we need to issue a certificate. To do this, we will be using `cert-manager`.

Install certbot by running the following command:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
```

Verify that it has been deployed successfully:

```bash
kubectl get deployments -n cert-manager
```

Verify that the pods are running:

```bash
kubectl get pods -n cert-manager
```

Pods `cert-manager`, `cert-manager-cainjector` and `cert-manager-webhook` should all be in a running state.

### Create a Let's Encrypt ClusterIssuer

In order to get a certificate, we need to create a ClusterIssuer resource.

Save the YAML below into a file called `letsencrypt-clusterissuer.yaml` using your favourite text editor. Don't forget to replace `<your-email>` with your actual email.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory # production
    #server: https://acme-staging-v02.api.letsencrypt.org/directory # staging
    email: _YOUR_EMAIL_ # replace this with your email
    privateKeySecretRef:
      name: letsencrypt
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply the YAML by running the following:

```bash
kubectl apply -f letsencrypt-clusterissuer.yaml
```

### Update the application ingress to use SSL/TLS

In order for `cert-manager` to issue a certificate from Let's Encrypt, we need to update our ingress resource.

Our resource must contain the annotations `cert-manager.io/cluster-issuer: "letsencrypt"` and `kubernetes.io/ingress.class: nginx`. We also need to add a new `tls:` object inside `spec:` specifying the certificate location.

Use the following command to edit the ingress:

```bash
kubectl edit ingress gpip -n gpip
```

Edit the ingress so that it looks something like this:

> **Note:** Use the command `i` to enter text in `vim` (default shell editor). Press the escape key and enter command `:wq!` to save changes.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  # Add the following annotations:
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt"
  creationTimestamp: "2021-02-15T18:45:47Z"
  generation: 1
  name: gpip
  namespace: gpip
  resourceVersion: "183247"
  selfLink: /apis/extensions/v1beta1/namespaces/gpip/ingresses/gpip
  uid: 7f89a1f4-54ae-4ee3-8ebe-de195efdaf26
spec:
  # Add the TLS object:
  tls:
  - hosts:
    - gpip.<external-ip>.nip.io # same as previous host
    secretName: gpip-tls-secret
  # Leave the rest of the file intact
  rules:
  - host: gpip.<external-ip>.nip.io
    http:
      paths:
      - backend:
          serviceName: gpip
          servicePort: 80
        path: /
        pathType: Exact
status:
  loadBalancer:
    ingress:
    - ip: <external-ip>
```

After saving `cert-manager` will issue a certificate from Let´s Encrypt.

### Verify that the application uses SSL/TLS

Now you can verify that the application is reachable over HTTPS.

Run the following command to verify that TLS is enabled, and that a redirect is in place:

```bash
curl http://gpip.$external_ip.nip.io -vL
```

You should get an output similiar to the following:

```bash
*   Trying 51.124.87.220...
* TCP_NODELAY set
* Connected to gpip.51.124.87.220.nip.io (51.124.87.220) port 80 (#0)
> GET / HTTP/1.1
> Host: gpip.51.124.87.220.nip.io
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 308 Permanent Redirect
< Date: Mon, 15 Feb 2021 20:19:38 GMT
< Content-Type: text/html
< Content-Length: 164
< Connection: keep-alive
< Location: https://gpip.51.124.87.220.nip.io
< 
* Ignoring the response-body
* Connection #0 to host gpip.51.124.87.220.nip.io left intact
* Issue another request to this URL: 'https://gpip.51.124.87.220.nip.io/'
*   Trying 51.124.87.220...
* TCP_NODELAY set
* Connected to gpip.51.124.87.220.nip.io (51.124.87.220) port 443 (#1)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=gpip.51.124.87.220.nip.io
*  start date: Feb 15 19:13:47 2021 GMT
*  expire date: May 16 19:13:47 2021 GMT
*  subjectAltName: host "gpip.51.124.87.220.nip.io" matched cert's "gpip.51.124.87.220.nip.io"
*  issuer: C=US; O=Let's Encrypt; CN=R3
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7fc22f009200)
> GET / HTTP/2
> Host: gpip.51.124.87.220.nip.io
> User-Agent: curl/7.64.1
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 200 
< date: Mon, 15 Feb 2021 20:19:38 GMT
< content-type: application/json; charset=UTF-8
< content-length: 22
< strict-transport-security: max-age=15724800; includeSubDomains
< 
{"ip":"127.0.0.1"}
* Connection #1 to host gpip.51.124.87.220.nip.io left intact
* Closing connection 0
* Closing connection 1
```

## Summary

After finishing this lab, you should have achieved the following:

* Deployed `cert-manager` to your cluster.
* Issued a certificate for `gpip` from Let's Encrypt.
* Installed a certificate for your application's ingress resource.

## Next steps

Congratulations! You are done with the first session. Either clean up your resources, or keep them for the next session.
