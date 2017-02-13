+++
date = "2017-02-13T09:42:32+01:00"
draft = false
title = "Kubernetes nginx-ingress-controller"
menu = "main"
+++

#### Introduction

In this post I will explain, how I expose applications running on Kubernetes clusters to the internet with the
help of Ingress controllers.

But first a little bit about Kubernetes Ingresses and Services. On a very simplistic level a `Service` is a 
logical abstraction communication layer to pods. During normal operations pods get's created, destroyed, scaled out, etc.

A `Service` make's it easy to always connect to the pods by connecting to their `service` which stays stable during the
pod life cycle. A important thing about `services` are what their type is, it determines how the `service` expose itself
to the cluster or the internet. Some of the `service` types are :
 
 * ClusterIP
   Your service is only expose internally to the cluster on the internal cluster IP. A example would be to 
   deploy Hasicorp's vault and expose it only internally.
   
 * NodePort
   Expose the service on the EC2 Instance on the specified port. This will be exposed to the internet. Off course it 
   this all depends on your AWS Security group / VPN rules.
   
 * LoadBalancer
   Supported on Amazon and Google cloud, this creates the cloud providers your using load balancer. So on Amazon it creates
   a ELB that points to your `service` on your cluster. 
 
 * ExternalName
   Create a CNAME dns record to a external domain. 
 
For more information about `Services` look at https://kubernetes.io/docs/user-guide/services/

A `Ingress` is rules on how to access a `Service` from the internet.

For more information about `Ingresses` look at
https://kubernetes.io/docs/user-guide/ingress/

#### Scenario

Imagine this scenario, you have a cluster running, on Amazon, you have multiple applications deployed to it, some are API's some
are Java applications running inside something like Tomcat, and to add to the mix, you have a couple of static html pages sitting in a 
Apache web server that serves as documentation for your api's. All applications needs to have SSL, some of the api's endpoints have changed,
but you still have to serve the old endpoint path, so you need to do some sort of path rewrite. How do you expose everything to the 
internet? The obvious answer is create a  `type LoadBalancer` service for each, but, then multiple ELB's will be created, you have to 
deal with SSL termination at each ELB, you have to CNAME your applications/api's domain names to the right ELB's, and in general just have
very little control over the ELB. 

Enter `Ingress Controllers`. You deploy a `ingress controller`, create a `type LoadBalancer` service for it, and it sits and monitors
Kubernetes api server's /ingresses endpoint and acts as a reverse proxy for the ingress rules it found there. You then deploy
your application and expose it's service as a `type NodePort`, and create ingress rules for it. The `ingress controller` then
picks up the new deployed service and proxy traffic to it from outside.

Following this setup, you only have one ELB then on Amazon, and a central place at the `ingress controller` to manage the traffic coming
into your cluster to your applications.

#### Ingress controllers
Two of the more popular ones that I normally use, are Traefik and Nginx-ingress-controller. I like both, and for simple projects I prefer using Traefik
as it' easy to setup and use. However as the projects needs grow or for more complex projects, I reach for the nginx-ingress-controller.

Couple of differences between the two, that normally helps me decide which one to pick for a project.

* Traefik routes to the Kubernetes service, Nginx, by pass the service and routes to the pods. This allows nginx
  to do sticky sessions if your application needs it.
* Traefik works with multiple backends (docker, swarm, marathon, etcd, consul, kubernetes, etc. etc.) See https://docs.traefik.io/ 
  The nginx-ingress-controller is specfic for Kubernetes, you can off course build your own Nginx reverse proxy, perhaps with OpenResty
  and get it to work with almost any backend, but that requires a significant time investment.
* The nginx-ingress-controller can handle websockets, Traefik does not.
* Traefik can do basic path manipulation by using `PathPrefixStrip` but cannot do more complex path rewrites like Nginx.
* The nginx-ingress-controller is build on top of OpenResty (https://openresty.org). Using Lua you can easily extend Nginx capabilities
  and mold it to do whatever you need it to do.

For more information look at :

https://traefik.io/
https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx

In general I prefer the nginx-ingress-controller, so for the rest of the article I will focus on it.

#### Setup

Let's setup a little demo 'hello world' api on our  cluster and expose the api on the internet using 
nginx-ingress-controller. Let's deploy the `nginx-ingress-controller` first.

For the controller, the first thing we need to do is setup a default backend service for
nginx. 

The `default backend` is the default service that nginx falls backs to if if cannot route a request
successfully. The `default backend` needs to satisfy the following two requirements :

* serves a 404 page at /
* serves 200 on a /healthz

See more here https://github.com/kubernetes/contrib/tree/master/404-server

Let's use the example `default backend` from the nginx-ingress-controller github project

```
kubectl create -f https://raw.githubusercontent.com/kubernetes/contrib/master/ingress/controllers/nginx/examples/default-backend.yaml
```

If we want to handle SSL requests (which we should always do) Nginx needs to have a default SSL certificate. This
get's used for requests for there is not specified SSL certificate.

Assuming you have SSL cert and key, create `secrets` as follow, where tsl.key is the key name
and tsl.crt is your certificate and dhparam.pem is the pem file.
```
kubectl create secret tls tls-certificate --key tls.key --cert tls.crt 
kubectl create secret generic tls-dhparam --from-file=dhparam.pem 
```

Save the following to a file (e.g. nginx-ingress-controller.yml) :
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: LoadBalancer
  ports:
    - port: 80
      name: http
    - port: 443
      name: https
  selector:
    k8s-app: nginx-ingress-lb
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 2
  revisionHistoryLimit: 3
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: nginx-ingress-controller
          image: gcr.io/google_containers/nginx-ingress-controller:0.8.3
          imagePullPolicy: Always
          readinessProbe:
            httpGet:
              path: /healthz
              port: 18080
              scheme: HTTP
          livenessProbe:
            httpGet:
              path: /healthz
              port: 18080
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 5
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --default-ssl-certificate=$(POD_NAMESPACE)/tls-certificate
          # Use downward API
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 80
            - containerPort: 443
          volumeMounts:
            - name: tls-dhparam-vol
              mountPath: /etc/nginx-ssl/dhparam
            - name: nginx-template-volume
              mountPath: /etc/nginx/template
              readOnly: true
      volumes:
        - name: tls-dhparam-vol
          secret:
            secretName: tls-dhparam
        - name: nginx-template-volume
          configMap:
            name: nginx-template
            items:
            - key: nginx.tmpl
              path: nginx.tmpl
```

You might have noticed as well, we specify a `default-backend-service`, this is used by nginx to route requests on 
which it found not ingress rule match. We need to deploy this `default-backend-service` before we deploy
nginx. We can use the example default backend yaml file from the Kubernetes ingress github repo. After the deployment
expose the `default-http-backend` so that the nginx-ingress-controller can communicate with it.

```
kubectl create -f https://raw.githubusercontent.com/kubernetes/contrib/master/ingress/controllers/nginx/examples/default-backend.yaml
kubectl expose rc default-http-backend --port=80 --target-port=8080 --name=default-http-backend
```

We can now deploy `nginx-ingress-controller`. 

```
kubectl create -f ./nginx-ingress-controller.yml
```

What this command did was, it created a deployment with two replicas of the `nginx-ingress-controller`
and a service for it of `type LoadBalancer` which created a ELB for us on AWS.
Let's confirm that. Get the service :

```
kubectl get services -o wide | grep nginx
```

You should get something similar to the following :
```
nginx-ingress  100.62.254.11  aaba00ec3d35211r68caa0a32e7f202e-566d19265.eu-west-1.elb.amazonaws.com  80:32199/TCP,443:31772/TCP 25d  k8s-app=nginx-ingress-lb
```

This means the ELB on Amazon got created. Any domain's who's traffic we want to go to this 
ingress controller should be CNAMED to this ELB hostname (`aaba00ec3d35211r68caa0a32e7f202e-566d19265.eu-west-1.elb.amazonaws.com`)
For the rest of this post, I am going to assume api.daemonza.io is CNAMED to this ELB

Make sure the deployment pods are up and running :
```
kubectl get pods | grep nginx
```

You should get something similar to the following : 
```
nginx-ingress-controller-1461575992-62j00   1/1       Running   0          30sec
nginx-ingress-controller-1461575992-eoqqb   1/1       Running   0          30sec
```

Using the ELB hostname that we got from the querying for the `nginx-ingress-controller` service, make
sure that traffic is being routed.

```
curl -v aaba00ec3d35211r68caa0a32e7f202e-566d19265.eu-west-1.elb.amazonaws.com
```
You should see the following :

```
HTTP/1.1 404 Not Found
Server: nginx/1.11.3
```

and the response should be 

```
default backend - 404
```

This means everything is working correctly and the ELB forwarded traffic to our
`nginx-ingress-controller` and the `nginx-ingress-controller` passed it along to the 
`default-backend-service` that we deployed.

#### Deploying our API application

I like Go, so I created a little API that we can deploy and test the ingress controller with.
This is the api we are going to deploy :
https://hub.docker.com/r/daemonza/testapi/
and the super simplistic source code is at https://github.com/daemonza/testapi 

Create a file, let's call it testapi.md with the following contents
```
apiVersion: extensions/v1beta1 
kind: Deployment 
metadata:
  name: testapi 
  namespace: default 
  labels:
    application: testapi
spec:
  replicas: 3
  selector:
    matchLabels:
      application: testapi
  template:
    metadata:
      labels:
        application: testapi 
    spec:
      containers:
        - name: testapi
          image: docker pull daemonza/testapi:latest
          imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: testapi 
  namespace: default 
  labels:
    application: testapi 
spec:
  type: NodePort
  selector:
    application: testapi 
  ports:
  - port: 8080
    targetPort: 8080 
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: testapi
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: "api.daemonza.io"
    http:
      paths:
      - path: /testapi
        backend:
          serviceName: testapi
          servicePort: 8080
```

Quick run through of the above file. It consists of three parts, deployment, service, ingress.

I normally split these into threes separate files, but for the sake of this blog post, let's
keep it all in one file. For the nginx-ingress-controller we are mostly interested in
the `ingress` part.

In the form of `annotations` it's possible for us to pass some configuration through to the nginx-ingress-controller.
In the above example we say we want this to always be used with nginx-ingress-controller, in case we have multiple
controllers, and we always want to redirect HTTP to HTTPS.
Then we want to route all traffic where the host (virtual host) is api.daemonza.io but only on the `/testapi` path, and
we want to route the traffic to the `testapi service`, which in turn expose the `testapi deployment`.

If we look at the equivalent simplified nginx configuration it would look similar to the following, where the `upstream` is
`deployment` with three replica's.

```
    upstream default-testapi-8080 {
        server 100.96.3.148:8080 max_fails=0 fail_timeout=0;
        server 100.96.4.222:8080 max_fails=0 fail_timeout=0;
        server 100.96.4.223:8080 max_fails=0 fail_timeout=0;   
    }
    
    server {
            server_name api.daemonza.io;
            listen 80;

            location /testapi {
                   
                proxy_pass http://default-testapi-8080;
            }
            
```

Now let's test it.
Deploy with the following command
```
kubectl create -f ./testapi.md
```

You should see output similar to 

```
deployment "testapi" created
service "testapi" created
ingress "testapi" created
```

Let's check on the ingress rule

```
kubectl get ingress testapi
```

You should see something similar to :

```
NAME      HOSTS             ADDRESS          PORTS     AGE
testapi   api.daemonza.io   51.232.218.155   80
```

Let's ask for a little more information with `describe`
```
kubectl describe ingress testapi


Name:			testapi
Namespace:		default
Address:		52.214.207.195
Default backend:	default-http-backend:80 (<none>)
Rules:
  Host			Path	Backends
  ----			----	--------
  api.daemonza.io
    			/testapi 	testapi:8080 (<none>)
Annotations:
  ssl-redirect:	true
Events:
  FirstSeen	LastSeen	Count	From				SubObjectPath	Type		Reason	Message
  ---------	--------	-----	----				-------------	--------	------	-------
  2m		2m		1	{nginx-ingress-controller }			Normal		CREATE	default/testapi
  2m		2m		1	{nginx-ingress-controller }			Normal		CREATE	ip: 51.232.218.155
  2m		2m		1	{nginx-ingress-controller }			Normal		UPDATE	default/testapi
```

From this we can see that we asked the nginx-ingress-controller to route requests
for `api.daemonza.io` on path `/testapi` to our backend `testapi`. Plain HTTP requests
will be redirected over to HTTPS and the testapi is scaled out to three replicas.

Ok so with the testapi deployed, let's see if everything works.
The test api expose the following endpoints :
```
PUT /put/:something
GET /get/:something
POST /post/:something
DELETE /get/:something
PATCH /patch/:something
```
Where :something is anything you want to put there.

Let's try a POST request
```
curl -X POST http://api.daemonza.io/post/hello
```

We should get http status code 301(moved permanently) back, this is because in our
testapi ingress specification, we added this annotation :

```
ingress.kubernetes.io/ssl-redirect: "true"
```

Which tells nginx to redirect http to https.
If we do the request again on https we should get status 200 back. Assuming that worked, let's move on and add
some virtual hosts for our testapi.

#### Virtual hosts

Change the testapi.md ingress specification to look as follow, change the host domain name to a domain that you own
or setup them up in your local /etc/hosts file for testing.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: testapi
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: "blue.daemonza.io"
    http:
      paths:
      - path: /testapi
        backend:
          serviceName: testapi
          servicePort: 8080  
  - host: "green.daemonza.io"
    http:
      paths:
      - path: /testapi
        backend:
          serviceName: testapi
          servicePort: 8080  
  - host: "api.myfakedomain.io"
    http:
      paths:
      - path: /testapi
        backend:
          serviceName: testapi
          servicePort: 8080
```

You can also route multiple virtual hosts on different paths to the same backend service.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: testapi
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: "blue.daemonza.io"
    http:
      paths:
      - path: /testapi/get/blue
        backend:
          serviceName: testapi
          servicePort: 8080
  - host: "green.daemonza.io"
    http:
      paths:
      - path: /testapi/get/green
        backend:
          serviceName: testapi
          servicePort: 8080                 
```

In the above the example we are routing two different virtual hosts, to the same API, but on different
api endpoints. We can then off course also do the opposite by having multiple little api "micro services"
and route traffic to the correct one by matching up the backend serviceName with the Path.

Let's test requests to our api on the api.myfakedomain.io virtual host we added.
Add api.myfakedomain.io to your /etc/hosts file in order to test with that domain.
```
curl https://api.myfakedomain.io
```

Running that curl command, curls tells us `curl: (60) SSL certificate problem: Invalid certificate chain`
which is correct as we don't have the SSL certificate for that domain configured in nginx. Luckily through
ingress rules, we can specify a ssl certificate per virtual host as follow :

First thing we need to do is create a Kubernetes secret containing our SSL certificate and key

Add the following to a file, let's call is apitest_ssl.md
```
apiVersion: v1
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
kind: Secret
metadata:
  name: apitestsecret
  namespace: default
type: Opaque
```

Then deploy it to Kubernetes with
```
kubectl create -f ./apitest_ssl.md
```

Modify the apitest.md file ingress specification for api.myfakedomain.io to reference this secret as follow :

```
spec:
  tls:
  - hosts:
    - "api.myfakedomain.io"
    secretName: apitestsecret
  rules:
  - host: "api.myfakedomain.io"
    http:
      paths:
      - path: /testapi
        backend:
          serviceName: testapi
          servicePort: 8080
```

Now if you ask for the api.myfakedomain.io ingress rules

```
kubectl get ingress api.myfakedomain.io
```

you should get a similar reply to the following :

```
Name:			api.myfakedomain.io
Namespace:		default
Address:		52.223.113.173,52.224.52.153
Default backend:	default-http-backend:80 (<none>)
TLS:
  apitestsecret terminates api.myfakedomain.io
Rules:
  Host			Path	Backends
  ----			----	--------
  api.myfakedomain.io
    			/testapi 	apitest:8080 (<none>)
Annotations:
  ssl-redirect:	true
```

Running the curl command now against https://api.fakedomain.io should work without any warnings about
insecure certificates.

#### Path rewrites

Sometimes, there is a need to rewrite the path of a request to match up with the backend service. One such scenario might be, 
a API got developed and deployed, got changed over time, but there is still a need to be backwards
compatibility on the API endpoints.

So using out deployed apitest api, lets setup a path rewrite. We have a `/new/get` api path, but the api does
not support it, it supports, `/get/new` we now need to rewrite the path to that. Using a annotation with the ingress rules 
we can rewrite the path as follow :

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: apitest
  namespace: applications
  annotations:
    ingress.kubernetes.io/rewrite-target: /new/get
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: "api.daemonza.io"
    http:
      paths:
      - path: /get/new
        backend:
          serviceName: apitest
          servicePort: 8080
```


#### Sticky sessions

Won't it be nice if all applications got developed from the start with the 12-factor (https://12factor.net/) methodology
in mind? Unfortunately that does not always happen, and there is times when you have to be able
to handle stateful application on your Kubernetes cluster.

Luckily for us the nginx-ingress-controller can handle sticky sessions as it bypass the service level and 
route directly the pods.

In order to get sticky sessions to work we need to create a configMap and enable it.

Save the following in a file, for now call it nginx.conf.md
```
apiVersion: v1
data:
  enable-sticky-sessions: "true"
kind: ConfigMap
metadata:
  name: nginx-ingress-controller-conf
```

and load it with

```
kubectl create -f ./nginx.conf.md
```

The controller will reload on configuration change.

What this setting does it, instruct nginx to use the nginx-sticky-module-ng module
(https://bitbucket.org/nginx-goodies/nginx-sticky-module-ng) that's bundled with the controller
to handle all sticky sessions for us.

This is how the generated nginx configuration looks after enabling sticky sessions.

```
upstream default-testapi-8080 {
    sticky name=SERVERID httponly path=/;
    server 100.96.3.148:8080 max_fails=0 fail_timeout=0;
    server 100.96.4.222:8080 max_fails=0 fail_timeout=0;
    server 100.96.4.223:8080 max_fails=0 fail_timeout=0;   
}
    
server {
        server_name api.daemonza.io;
        listen 80;

        location /testapi {           
            proxy_pass http://default-testapi-8080;
        }
```


#### Proxy protocol 

Lots of times you need to pass a user's IP address / hostname through to your application. A example would 
be, to have the hostname of the user in your application logs.

By default if you deploy the nginx-ingress-controller on AWS behind a ELB, the ELB will not pass along the hostname
information, to solve this we need to enable `proxy protocol`.

Add the following to your Nginx configMap.
```
use-proxy-protocol: "true"
```

And in your nginx-ingress-controler service specification add the following annotation

```
service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: '*'
```
This will make sure that the ELB that get's created will have `proxy protocol` enabled.
If you prefer not to change the ELB from a Kubernetes Service, you can configure it manually on the ELB
by following the documentation here : http://docs.aws.amazon.com/elasticloadbalancing/latest/classic/enable-proxy-protocol.html

And excellent explanation of what proxy protocol is and how it works can be found at
http://www.haproxy.org/download/1.8/doc/proxy-protocol.txt


#### Custom Nginx configuration

Sometimes there is a need to configure something in Nginx, that's not possible through the nginx-ingress-controller
configMap, annotations or ingress rules. In such cases, it's possible to edit `nginx.tmpl` Go template.

You can get the nginx.tmpl file from the nginx-ingress-controller Github repository or, copy it over from your
nginx-ingress-controller pod. It's located at `/etc/nginx/template/nginx.tmpl`. Make your changes to it
and then save it as a configMap as follow

```
kubectl create configmap nginx-template --from-file=nginx.tmpl=./nginx.tmpl
```
 
then mount the nginx-template configMap as a volume in the your nginx-ingress-controller
specification.

```
volumeMounts:
   - name: nginx-template-volume
     mountPath: /etc/nginx/template
      readOnly: true
volumes:
   - name: nginx-template-volume
     configMap:
       name: nginx-template
       items:
       - key: nginx.tmpl
         path: nginx.tmpl
```

#### Conclusion

The Kubernetes ingress specifications combined with the nginx-ingress-controller gives a incredible flexible and
powerful routing platform for your Kubernetes clusters. For more information about Kubernetes Ingress and the
Nginx-ingress-controller visit :

https://kubernetes.io/docs/user-guide/ingress/
https://github.com/nginxinc/kubernetes-ingress/tree/master/nginx-controller