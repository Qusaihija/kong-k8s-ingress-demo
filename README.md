# kong-k8s-ingress-demo

This project demonstrates three different methods to deploy an application behind a Kong Ingress Controller in a Kubernetes cluster, showcasing varying levels of security and configuration complexity.

## Overview

The repository contains three Kubernetes manifest files, each defining a complete setup for a specific use case:

1.  **`without_ssl.yaml`**: A basic setup with Kong API key authentication but no TLS encryption.
2.  **`kong_ssl_issuer.yaml`**: Adds TLS termination with a Let's Encrypt certificate using a namespaced `Issuer` (for use within a single namespace).
3.  **`kong_ssl_clusterissuer.yaml`**: Adds TLS termination with a Let's Encrypt certificate using a cluster-wide `ClusterIssuer` (for use across multiple namespaces).

## Prerequisites

Before applying any of these manifests, ensure you have the following installed and configured in your Kubernetes cluster:

1.  **Kong Ingress Controller**: The Kong Ingress Controller for Kubernetes must be installed.
    *   Installation guide:
    ```
    kubectl create namespace kong
    helm repo add kong https://charts.konghq.com
    helm repo update
    helm install kong kong/kong --namespace kong --set ingressController.installCRDs=false
    ```
2.  **cert-manager** (Required for SSL configurations only):
    ```
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.1/cert-manager.crds.yaml
    kubectl create namespace cert-manager
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    helm install cert-manager jetstack/cert-manager \
        --namespace cert-manager \
        --version v1.15.1
    ```

3.  **A Domain Name**: You must own a domain name and have the ability to point its DNS `A` record to the public IP address of your application

## File Descriptions & Usage

### 1. Basic Setup (No TLS) - `without_ssl.yaml`

This is the simplest configuration. It sets up the application with Kong's key-auth plugin but does not encrypt traffic with HTTPS.

**Features:**
*   Flask application deployment and service.
*   Kong key-auth plugin requiring an API key.
*   A `KongConsumer` and `Secret` credential.
*   Kong Ingress routing HTTP traffic.

**Steps to Apply:**
1.  Update the `host` field in the Ingress rule from `justctf.xyz` to your domain.
2.  Apply the manifest:
    ```bash
    kubectl apply -f without_ssl.yaml
    ```
3.  Test the endpoint. You will need to provide the API key in the `username` header.
    ```bash
    curl -H "username: omniops" http://justctf.xyz
    ```

---

### 2. With TLS using an Issuer - `kong_ssl_issuer.yaml`

This configuration adds automatic TLS certificate management using cert-manager's `Issuer` resource, which is scoped to a single namespace.

**Features:**
*   Everything from the basic setup.
*   A namespaced `Issuer` resource for Let's Encrypt.
*   A `Certificate` resource that requests a TLS cert.
*   Ingress configured to use TLS and the cert-manager issuer.

**Steps to Apply:**
1.  **Crucial:** Replace `your-email@example.com` in the `Issuer` spec with a valid email address for Let's Encrypt notifications.
2.  Update all instances of the domain `justctf.xyz` in the `Certificate` and `Ingress` to your domain.
3.  Apply the manifest:
    ```bash
    kubectl apply -f kong_ssl_issuer.yaml
    ```
4.  You can check the status of the certificate issuance with:
    ```bash
    kubectl get certificate -n omniops-demo
    ```
5.  Once the certificate is `Ready`, test the secure endpoint:
    ```bash
    curl -H "username: omniops" https://justctf.xyz
    ```

---

### 3. With TLS using a ClusterIssuer - `kong_ssl_clusterissuer.yaml`

This configuration uses a `ClusterIssuer`, which is a cluster-scoped resource. This is the recommended approach if you plan to manage certificates for multiple applications across different namespaces.

**Features:**
*   Everything from the basic setup.
*   A cluster-scoped `ClusterIssuer` resource for Let's Encrypt.
*   A `Certificate` resource that requests a TLS cert, referencing the `ClusterIssuer`.
*   Ingress configured to use TLS and the cert-manager cluster issuer.

**Steps to Apply:**
1.  **Crucial:** Replace `your-email@example.com` in the `ClusterIssuer` spec with a valid email address.
2.  Update all instances of the domain `justctf.xyz` in the `Certificate` and `Ingress` to your domain.
3.  Apply the manifest:
    ```bash
    kubectl apply -f kong_ssl_clusterissuer.yaml
    ```
4.  Check the certificate status:
    ```bash
    kubectl get certificate -n omniops-demo
    ```
5.  Test the secure endpoint:
    ```bash
    curl -H "username: omniops" https://justctf.xyz
    ```

## Key Components Explained

*   **Namespace (`omniops-demo`)**: Isolates all resources for this demo.
*   **Deployment & Service**: Runs the `0xnef/flask-kong-app` container and exposes it within the cluster (changable if you want).
*   **Secret (`omniops-apikey`)**: Stores the API key credential with a special label (`konghq.com/credential`) that Kong can recognize.
*   **KongConsumer**: Defines a consumer (`omniops`) and links it to the credential secret.
*   **KongPlugin**: Configures the `key-auth` plugin to require a key in the `username` header and to hide the credential from upstream services.
*   **Issuer/ClusterIssuer**: Defines cert-manager's configuration to issue certificates from Let's Encrypt's production ACME server.
*   **Certificate**: Requests a specific TLS certificate for a domain, which will be stored in a `Secret`.
*   **Ingress**: Routes external HTTP/HTTPS traffic to the Flask service and attaches the auth plugin and TLS certificate.

## Authentication

All configurations enforce API key authentication. To access the service, you must include the key in the `username` header.

**Example:**
```bash
# For HTTP (without_ssl.yaml)
curl -H "username: omniops" http://YOUR_DOMAIN

# For HTTPS (ssl configurations)
curl -H "username: omniops" https://YOUR_DOMAIN
```

## Troubleshooting
* **Certificate not issuing**:

    Check cert-manager logs: ```kubectl logs -f deployment/cert-manager -n cert-manager```

    Describe the Certificate resource: ```kubectl describe certificate justctf-xyz-cert -n omniops-demo```

* **HTTP 401 Unauthorized**:

    Verify the username header is spelled correctly.

    Verify the value matches the key in the omniops-apikey secret.

* **HTTP 404 Not Found / Service Not Reachable**:

    Verify the Kong Ingress Controller is running: kubectl get pods -n kong

    Check the external IP of the kong-proxy service: kubectl get svc -n kong

    Describe the Ingress resource: kubectl describe ingress flask-ingress -n omniops-demo




