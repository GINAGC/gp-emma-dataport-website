# Reverse proxy setup for https://dataport.emma.msrb.org

## Requirements

- **A kubernetes cluster running**

- **kubectl tool**

- **Docker**

- **Git**

## Clone the needed repositories
```
git clone https://github.com/filetrust/icap-infrastructure.git
git clone https://github.com/k8-proxy/k8-reverse-proxy.git
git clone https://github.com/k8-proxy/s-k8-proxy-rebuild.git
git clone https://github.com/k8-proxy/gp-emma-dataport-website.git
```

## Installation

- Install ICAP server and adaptation service
  
```
cd icap-infrastructure/adaptation
```

Create the Kubernetes namespace
```
kubectl create ns icap-adaptation
```

Create container registry secret
```
kubectl create -n icap-adaptation secret docker-registry regcred	\ 
	--docker-server=https://index.docker.io/v1/ 	\
	--docker-username=<username>	\
	--docker-password=<password>	\
	--docker-email=<email address>
```

copy the updated helm template file mvp-icap-service-configmap.yml from gp-emma-dataport-website repo to the templates folder 
```
cp ../../gp-emma-dataport-website/patch/mvp-icap-service-configmap.yml templates/
```

Install the cluster components
```
helm install . --namespace icap-adaptation --generate-name
```

The cluster's services should now be deployed
```
> kubectl get pods -n icap-adaptation
NAME                                 READY   STATUS    RESTARTS   AGE
adaptation-service-64cc49f99-kwfp6   1/1     Running   0          3m22s
mvp-icap-service-b7ddccb9-gf4z6      1/1     Running   0          3m22s
rabbitmq-controller-747n4            1/1     Running   0          3m22s
```
 
- Setup squid icap client and nginx for reverse proxy
  
Switch to the reverse proxy build repo
  ```bash
    cd ../../k8-reverse-proxy/stable-src/
  ```

Build and push the needed images to your dockerhub registry
  ```bash
    docker build nginx -t <docker registry>/reverse-proxy-nginx:0.0.1
    docker push <docker registry>/reverse-proxy-nginx:0.0.1

    docker build squid -t <docker registry>/reverse-proxy-squid:0.0.1
    docker push <docker registry>/reverse-proxy-squid:0.0.1
  ```

Switch to the reverse proxy k8s deployment repo
  ```bash
    cd ../../s-k8-proxy-rebuild/stable-src/
  ```

Record the icap server service IP address (it seems like squid doesn't like when we use the service name, still investigating that)
  ```bash
    $ kubectl -n icap-adaptation get svc | grep icap-service
    icap-service                        NodePort       10.4.6.142   <none>          1344:32278/TCP   23h
  ```
For the example above, ip is 10.4.6.142 


Setup the services
  ```bash
    helm upgrade --namespace icap-adaptation upgrade --install \
	--set image.nginx.repository=<docker registry>/reverse-proxy-nginx \
	--set image.nginx.tag=0.0.1 \
	--set image.squid.repository=<docker registry>/reverse-proxy-squid \
	--set image.squid.tag=0.0.1 \
	--set image.icap.repository=<docker registry>/reverse-proxy-c-icap \
	--set image.icap.tag=0.0.1 \
	--set application.nginx.env.ALLOWED_DOMAINS='dataport.emma.msrb.org.glasswall-icap.com\,www.dataport.emma.msrb.org.glasswall-icap.com' \
	--set application.nginx.env.ROOT_DOMAIN='glasswall-icap.com' \
	--set application.nginx.env.SUBFILTER_ENV='dataport.emma.msrb.org\,dataport.emma.msrb.org.glasswall-icap.com' \
	--set application.squid.env.ALLOWED_DOMAINS='dataport.emma.msrb.org.glasswall-icap.com\,www.dataport.emma.msrb.org.glasswall-icap.com' \
	--set application.squid.env.ROOT_DOMAIN='glasswall-icap.com' \
	reverse-proxy chart/
  ```

Edit the nginx deployment to set the correct icap server url (we need to use the icap server from adaptation service)
The server url should be : icap://<ip_recorded above>:1344/gw_rebuild
  ```bash
    $ kubectl -n icap-adaptation edit deployment/reverse-proxy-nginx
  ```

Edit the squid deployment to set the correct icap server url (we need to use the icap server from adaptation service)
The server url should be : icap://<ip_recorded above>:1344/gw_rebuild
  ```bash
    $ kubectl -n icap-adaptation edit deployment/reverse-proxy-quid
  ```

Setup port forward to be able to test from your host
 ```bash
    $ kubectl -n icap-adaptation port-forward svc/reverse-proxy-reverse-proxy-nginx 443:443
  ```  

Add the following line in your host file and test the setup
 ```
    127.0.0.1	 dataport.emma.msrb.org.glasswall-icap.com www.dataport.emma.msrb.org.glasswall-icap.com assets.publishing.service.gov.uk.glasswall-icap.com
  ```  



