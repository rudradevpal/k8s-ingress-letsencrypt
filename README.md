# Securing k8s ingress with *cert-manager* & *Let's Encrypt*
### Ingress
**Ingress** provides balancing, SSL termination and name-based routing to Kubernetes services. For more details visit [official page](https://kubernetes.io/docs/concepts/services-networking/ingress)
### Let's Encrypt
**Let's Encrypt** is a Security Research Group & TLS Certificate Authority. They provide X.509 certificates for Transport Layer Security encryption free of cost. But the validity of the certificate is 90 day. So for every 90 days, you need to renew the license.
### cert-manager
**cert-manager** is Kubernetes certificate management controller. It can help with issuing certificates from a variety of sources like *Let’s Encrypt*, *HashiCorp Vault*, *Venafi*, a simple signing keypair or self-signed. This controller also checks the if certificates are up to date, if not it renews the certificates.
The Certificate can be issued in two ways:
* DNS Validation (*Going to describe*)
* HTTP Validation

## Description
Here we will install Kubernetes Ingress Controller (NGINX), cert-manager and deploy an example application. For the example application, we will generate a certificate from Let's Encrypt and add it to ingress resource for the example application.

## Prerequisite
* Valid domain (For production use)
* Kubernetes Cluster or Minikube
* Static IP for Kubernetes Host
* CloudFlare account configured with Domain (For DNS Validation Procedure. Check [this](https://support.cloudflare.com/hc/en-us/articles/201720164-Creating-a-Cloudflare-account-and-adding-a-website#h_8ea0d664-3af3-4623-ae2d-052f98b10090) for configuring domain with CloudFlare)

## Installation
1. Deploy [Nginx-Ingress](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml) Controller.
    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
    ```
2. Create NodePort of Ingress Controller Service
    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
    ```
3. Create Namespace for cert-manager
    ```shell
    kubectl create namespace cert-manager
    ```
4. Disable resource validation on the namespace that cert-manager runs in
    ```shell
    kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
    ```
5. Deploy cert-manager and CustomResourceDefinitions on Kubernetes
    ```shell
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.10.0/cert-manager.yaml
    ``` 
6. Add wildcard Entry in CloudFlare for your all Sub-Domain which will be pointing to Kubernetes Node IP where we have created *Ingress Controller Service NodePort* in step 2.
7. Get your CloudFlare Global API Key from the CloudFlare Dashboard
    * Goto CloudFlare Dashboard
    * Select Domain
    * Goto Account Setting
    * Click Get your API key link
    * Click View next to Global API Key
    * Enter your details and pass the I am not a robot challenge and click View
8. Set ENV `API_KEY` with the API Key by encoding to base64. Run Following command to create a secret:
    ```shell
    API_KEY=$(echo -n 3f7cf4643a2cfeddcadf16ea1ee92f40b1fa7 | base64 -)
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-api-key
      namespace: cert-manager
    type: Opaque
    data:
      api-key.txt: ${API_KEY}
    EOF
    ```
9. Set ENV `EMAIL_ADDRESS` with your Email Address and Run Following command to create a ClusterIssuer:
    ```shell
    EMAIL_ADDRESS="me.example@mymail.com"
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: ${EMAIL_ADDRESS}
        privateKeySecretRef:
          name: letsencrypt
        dns01:
          providers:
          - name: cf-dns
            cloudflare:
              email: ${EMAIL_ADDRESS}
              apiKeySecretRef:
                name: cloudflare-api-key
                key: api-key.txt
    EOF
    ```
10. Create a nginx application
    ```shell
    kubectl run nginx --image nginx -n default
    ```
11. Create Service for the nginx application
    ```shell
    kubectl expose nginx --port 80 -n default
    ```
12. Set ENV `DOMAIN_NAME` with your Subdomain Address you want to create for the application and Run Following command to create a Certificate. This Certificate will automatically create a secret with the same name of subdomain ('.' replaced by '-'). Also don't forget to change the `namespace` metadata if you are using different namespace for your application:
    ```shell
    DOMAIN_NAME="nginx.example.com"
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Certificate
    metadata:
      name: $(echo $DOMAIN_NAME | tr . -)
      namespace: default
    spec:
      secretName: $(echo $DOMAIN_NAME | tr . -)
      issuerRef:
        name: letsencrypt
        kind: ClusterIssuer
      commonName: '${DOMAIN_NAME}'
      dnsNames:
      - ${DOMAIN_NAME}
      acme:
        config:
        - dns01:
            provider: cf-dns
          domains:
            - ${DOMAIN_NAME}
    EOF
    ```
13. Check if Certificate Issued successfully. It may take few minutes.
    ```shell
    kubectl describe certificate  $(echo $DOMAIN_NAME | tr . -) -n default
    ```
14. Check if the certificate has created the `secret` successfully. You will get a secret with the same name of subdomain ('.' replaced by '-').
    ```shell
    kubectl get secret -n default
    ```
15. Create `ingress` resource for your application and use the `secret` generated by the `certificate`. Also, don't forget to change the `namespace` metadata if you are using a different namespace for your application.
    ```shell
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: nginx               # Name of Ingress Resource
      namespace: default        # Namespace of your application
      annotations:
        kubernetes.io/ingress.class: "nginx"
        # nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" # For https containernonly
        # certmanager.k8s.io/cluster-issuer: "letsencrypt"
        # nginx.ingress.kubernetes.io/ssl-redirect: "true"
        # nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    spec:
      tls:
      - hosts:
        - ${DOMAIN_NAME}
        secretName: $(echo $DOMAIN_NAME | tr . -)
      rules:
      - host: ${DOMAIN_NAME}
        http:
          paths:
          - path: /
            backend:
              serviceName: nginx    # Service Name of your Application
              servicePort: 80       # Service Port of your Application. If HTTPS uncomment line - 7
    EOF
    ```
16. To delete certificate and secret generated by certificate run:
    ```shell
    kubectl delete certificate  $(echo $DOMAIN_NAME | tr . -) -n default
    kubectl delete secret  $(echo $DOMAIN_NAME | tr . -) -n default
    ```
    **Don't delete and create same certificate for multiple times. Somtime it does not allow to create it for multiple time**
