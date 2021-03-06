Ingress
=======

Module objectives
-----------------

- serve app traffic from the ingress instead of LoadBalancer service
- use static IP with Ingress
- specify app domain
- add SSL support

---

Theory
------

Ingress is an API object that manages external access to the services in a cluster, typically HTTP.

Ingress can provide load balancing, SSL termination and name-based virtual hosting.

---
1. Change the frontend service type from LoadBalancer to NodePort
    ```
    $ kubectl edit svc/frontend
    ```
    Find the line type: LoadBalancer and change it to type: NodePort.

    Save the file Esc - :wq

    Ingress forwards traffic to a service (not directly to pods) That's why we still need a service wrapping our frontend pod. However, it is not necessary for this service to be of a LoadBalancer type, because now we will be accessing the frontend through ingress and not through a dedicated frontend load balancer. The service has to be of a NodePort type instead.

1. Check the service type

    ```
    $ kubectl get svc
    NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
    backend      ClusterIP   10.111.2.72    <none>        8080/TCP       5h
    db           ClusterIP   10.111.13.61   <none>        3306/TCP       1d
    frontend     NodePort    10.111.10.20   <none>        80:31661/TCP   36m
    kubernetes   ClusterIP   10.111.0.1     <none>        443/TCP        1d
    ```

1. Create file `k8s/training/ingress.yaml`

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: gceme-ingress
    spec:
      rules:
      - http:
          paths:
          - path: /gceme/*
            backend:
              serviceName: frontend
              servicePort: 80
    ```

    It will expose the service `frontend` using relative path `/gceme`.

1. Create the ingress

    ```
    # in the first terminal
    $ kubectl get ingress --watch
    NAME            HOSTS   ADDRESS   PORTS   AGE
    gceme-ingress   *                 80      0s
    gceme-ingress   *     35.227.223.114   80    6m22s
    ```

    ```
    # in the second terminal
    $ kubectl apply -f k8s/training/ingress.yaml
    ```

    Wait until you see IP in the address field. The application will be available as `http://<ingress-ip>/gceme/`

1. In the cloud console go to 'Network servcies' -> 'Load balancing' and examine the created load balancer.

Use static IP
-------------

By default, ingress uses ephemeral IP which may change during the time. To create DNS record and issue SSL certificates one needs static IP. In this exercise, you will create one and use it with ingress.

1. Create static IP

    ```
    $ gcloud compute addresses create web-static-ip --global
    ```

1. Assign it to the ingress

    ```
    $ kubectl edit ingress/gceme-ingress
    ..
    metadata:
      name: gceme-ingress
      annotations:
        kubernetes.io/ingress.global-static-ip-name: "web-static-ip"
    ```

    When you save the file Kubernetes will change the IP of the load balancer according to the annotation. You may get new IP from the Cloud Console or ingress resource.

Exercise 2 (Optional): Specify app domain
-----------------------------------------

1. Now, gceme app should be accessed using a specific DNS name.
1. Modify your `/etc/hosts` and set `gceme-training.com` domain to be resolved to the ingress IP address.
1. Modify ingress definition appropriately. Find section `Name-based virtual hosting` in [this](https://kubernetes.io/docs/concepts/services-networking/ingress/#types-of-ingress) document for reference.
1. Access `gceme-training.com` from your web browser.
1. Verify that you can't access `gceme` app using IP address anymore

Exercise 3 (Optional): Use TLS
------------------------------

1. Create a self-signed certificate for `gceme` app [link](https://stackoverflow.com/questions/10175812/how-to-create-a-self-signed-certificate-with-openssl) for the `gceme-training.com` domain
1. Create a secret for `gceme` app. The secret should contain the certificate and private key.
1. Add a `tls` section to the ingress definition. You can use the `tls` section from [this](https://kubernetes.io/docs/concepts/services-networking/ingress/#types-of-ingress) document for reference.
1. Redeploy, open app in a web browser and examine certificate details. Use [this](https://www.ssl2buy.com/wiki/how-to-view-ssl-certificate-details-on-chrome-56) link to see how a certificate can be viewed in chrome.
