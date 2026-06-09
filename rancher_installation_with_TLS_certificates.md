# Installing Rancher with TLS Certificates (Public CA, Private CA, and Self-Signed)

## What's the TL;DR:
 Rancher supports three TLS approaches:

  1. Public CA → simplest, fully trusted
  2. Private CA → requires CA bundle + Rancher configuration
  3. Rancher-generated (self-signed) → easiest, but not trusted by default

 TLS can be configured via:
  1. Helm CLI (--set)
  2. values.yaml (recommended for production)

 TLS Configuration Overview:
 
  | Parameter              | Description                        |
  | ---------------------- | ---------------------------------- |
  | `hostname`             | Rancher FQDN                       |
  | `ingress.tls.source`   | TLS source (`secret` or `rancher`) |
  | `privateCA`            | Enables private CA support         |
  | `additionalTrustedCAs` | Injects custom CA bundle           |

## Prerequisites:
 1. Kubernetes cluster is ready
 2. Helm installed and configured
 3. DNS record points to ingress/LB
 4. kubectl access to cluster

Add Rancher Helm Repository:

 helm repo add rancher https://releases.rancher.com/server-charts/latest
 
 helm repo update

## Option 1: Install Rancher with Public CA Certificate:
 When to Use
  1. Internet-facing deployments
  2. Certificates from trusted providers (Let’s Encrypt, DigiCert, etc.)
 
 1. Create TLS Secret
    
  kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=tls.crt --key=tls.key

 3. Installation using Helm:
    
  helm install rancher rancher/rancher --namespace cattle-system --create-namespace --set hostname=rancher.example.com --set ingress.tls.source=secret

 5. Installation using values.yaml:

 hostname: rancher.example.com
 ingress:
   tls:
     source: secret
 Below is the command to use after configuring the values.yaml above:
 
helm install rancher rancher/rancher -n cattle-system  -f values.yaml

✅ Expected Behavior
 1. Rancher uses the provided certificate
 2. No additional trust configuration required
 3. Browsers and agents trust the certificate automatically

## Option 2: Install Rancher with Private CA Certificate:
 When to Use
  1. Internal PKI environments
  2. Enterprise environments with custom CA
 Important
  Private CA is not trusted by default. You must configure Rancher and nodes accordingly.
 1. Create TLS Secret:
    
  kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=tls.crt --key=tls.key
 
 3. Create CA Bundle Secret:
    
  kubectl -n cattle-system create secret generic tls-ca --from-file=cacerts.pem=ca.crt

     Note: The file must include the full certificate chain (root + intermediates)
 5. Installation using Helm:
    
  helm install rancher rancher/rancher --namespace cattle-system --create-namespace --set hostname=rancher.example.com --set ingress.tls.source=secret --set privateCA=true --set additionalTrustedCAs=true

 7. Installation using values.yaml:
  hostname: rancher.example.com

  ingress:
    tls:
      source: secret

  privateCA: true
  additionalTrustedCAs: true
 
Below is the command to use after configuring the values.yaml above:

helm install rancher rancher/rancher -n cattle-system -f values.yaml

✅ Expected Behavior
 1. Rancher uses a private CA certificate
 2. Rancher injects CA into components
 3. Rancher Agents trust the connection 

## Option 3: Install Rancher with Rancher-Generated Self-Signed Certificate
 When to Use
  1. Lab / PoC environments
  2. Air-gapped bootstrap
  3. Quick deployments without PKI
 Limitations
  1. Not trusted by browsers by default
  2. Requires manual trust for agents
  3. Not recommended for production internet exposure

 1. Installation using Helm:
    
helm install rancher rancher/rancher --namespace cattle-system --create-namespace --set hostname=rancher.example.com --set ingress.tls.source=rancher
 3. Installation using values.yaml:
  hostname: rancher.example.com

  ingress:
    tls:
      source: rancher
  Below is the command to use after configuring the values.yaml above:
  
 helm install rancher rancher/rancher -n cattle-system -f values.yaml

Internal Behavior
Rancher generates:
  Self-signed CA
  Server certificate
Stored as Kubernetes secrets automatically





