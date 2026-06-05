# How to install FlashSystem.AI

## Introduction
This is documentation on how to install Flash.AI, a human language interpreter assistant that performs administrative tasks on IBM FlashSystem Storages. It should be noted that in this guide the container is run entirely on openshift in a local infrastructure, and integrates securely to both the IBM FlashSystem and the IBM Storage Insights systems.

## Required credentials
- From IBM Storage Insights:
  - Storage Insights API key
  - Storage Insights API key
- From IBM FlashSystem:
  - An administrator service account per FlashSystem
  - To use FlashSystem.AI, at least one system must be running version 9.1.2, and the others must be running version 9.1 or later. 
- From IBM Cloud:
  - IBM Cloud Login Credentials
  - IBM Cloud IAM API Key

## Openshift Environment Preparation
Generate an IAM API Key by accessing your IBM Cloud account and clicking on manage > Access (IAM) immediately after clicking on API Keys located on the sidebar. Configure the IAM API Key as you prefer. This will be your API <<YOUR_IAM_API_KEY>>.

## Creating a namespace
Log in via terminal to openshift and execute the following command:
``` 
oc create namespace <<Namespace_name>>
```
Wait for the message "namespace/<<Namespace_name>> created".

## Creating an image pull secret
This step performs an authentication of the openshifts in the IBM container registries, wait for the message "secret/icr-io created".
```
oc create secret docker-registry icr-io \
--docker-server=icr.io \
--docker-username=iamapikey \
--docker-password=<YOUR_IAM_API_KEY> \
--docker-email=<YOUR_EMAIL> \
-n <<nome_Namespace>>
```

# Preparing TLS certificates
The <<INGRESS_HOST>> is the DNS name at which the Flash.AI container will be accessed externally. To find the base domain use the command: oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}' This command should return something similar to: apps.mycluster.company.com And your <<INGRESS_HOST>> will be: <<Namespace_name>>.apps.mycluster.company.com 

## Frontend TLS
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout ingress.key -out ingress.crt \
-subj "/CN=<INGRESS_HOST>>"
```

```
oc create secret tls ingress-tls \
--key ./ingress.key --cert ./ingress.crt \
-n <<Namespace_name>>
```

## Backend TLS

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout application.key -out application.crt \
-subj "/CN=flashsystem-app-service.<<Namespace_name>>.svc.cluster.local"
```

```
oc create secret tls flashsystem-app-tls \
--key ./application.key --cert ./application.crt \
-n <<Namespace_name>>
```

# Preparing the Helm Chart
Log in inside the IBM container registries: 
```
helm registry login icr.io -u iamapikey
```
> **💡 The -u iamapikey needs to be inserted exactly as the command above shows! When prompting for the password insert your <<YOUR_IAM_API_KEY>>.**

## Pull the helm chart
Use the helm chart URL provided by IBM:
```
helm pull oci://icr.io/flashsystem-ai-release/flashsystem-ai:1.1.0
```
Extract the chart: 
```
tar -xvf flashsystem-ai-1.1.0.tgz
```
> **Note that in the directory you are currently in, the flashsystem-ai-1.1.0.tgz was extracted, enter the newly created folder with the ```cd flashsystem-ai``` command.**

# Configuring values.yaml
Edit the parameters of the values.yaml file as follows:

```
namespace:
  create: false
  name: "<<Namespace_name>>"
```

```
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: HTTPS
  route.openshift.io/termination: "reencrypt"
```

```
imagePullSecrets:
  - name: icr-io
```

```
tls:
  frontend:
    secretname: ingress-tls
  backend:
    secretname: flashsystem-app-tls
```

```
hosts:
  - host: <<INGRESS_HOST>>
    paths:
      - path: /
        pathType: Prefix
```

# Container Deploy
Deploy the container to the destination with the command: 
``` helm install flashsystem-ai . --namespace <<Namespace_name>> ```

> ### **For openshift environments execute the following command: Note: (whenever a new Route is made the command must be executed)**

```
oc patch route <<Openshift_route_name>> -n <<Namespace_name>> \
--type=merge \
-p "{\"spec\":{\"tls\":{\"destinationCACertificate\":\"$(awk '{printf \"%s\\n\",$0}' ~/application.crt)\"}}}"
```

# Accessing the configuration page

Open your browser and enter the address: "https://<<INGRESS_HOST>>/setup", you will come across the creation screen.

<img width="1894" height="970" alt="teladesenha " src="https://github.com/user-attachments/assets/ea688728-e201-42a8-8f3a-6a78429b7a8d" />

On the configuration page, locate the "GUI Key" section and insert the Tenant ID and an IBM Storage Insights REST API key, and click on regenerate to generate a GUI Key and click save.

<img width="1856" height="804" alt="image" src="https://github.com/user-attachments/assets/1094acbd-aae2-42ce-af62-c58c7e432c0f" />

Configure each FlashSystem on which you want to deploy Flash.ai. This includes systems running version 9.1.0 or later. Note: ”Only systems running version 9.1.2 displayed ChatBot."  Find the "Storage System" section and click on "Add storage system".
<img width="1892" height="488" alt="image" src="https://github.com/user-attachments/assets/372b1322-ddca-4080-ac65-3ad97ce1707b" />

Insert: System IP, user and password created exclusively for flash.ai

<img width="1390" height="702" alt="image" src="https://github.com/user-attachments/assets/f0aaaaba-ac1a-4e93-ae8d-67b2caa536d7" />

After clicking on "Test connection" it should be connected successfully.

<img width="1393" height="281" alt="image" src="https://github.com/user-attachments/assets/0a7e97b9-1845-4a32-875f-65ef53e6e7c6" />


Finally, connect to the FlashSystem via ssh and execute the command:
```
svctask chaicontainerinfo -key <<GUI_KEY>> -url <<INGRESS_HOST>>:443
svcinfo lsaicontainerinfo
```

Upon entering the IBM Storage Virtualize GUI, flash.ai will already be available.

<img width="1390" height="711" alt="image" src="https://github.com/user-attachments/assets/dad6e01a-3377-4ec0-8c78-edc1b32762d9" />
<img width="1390" height="715" alt="image" src="https://github.com/user-attachments/assets/d5f6275e-6039-4c8f-9a3e-657916d0db75" />
<img width="1390" height="711" alt="image" src="https://github.com/user-attachments/assets/c4071863-a9fa-4505-8e4e-995c9fc4f317" />

