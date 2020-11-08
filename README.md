# Kubernetes apigw
<br></br>

It's time to talk about the api gateway  

In a modern infrastructure , especially in a microservices environment you probably know what i'm referring to ,however it's better to clarify some points.

*An API gateway takes all API calls from clients, then routes them to the appropriate microservice with request routing, composition, and protocol translation. Typically it handles a request by invoking multiple microservices and aggregating the results, to determine the best path. It can translate between web protocols and web‑unfriendly protocols that are used internally.*

*An e‑commerce site might use an API gateway to provide mobile clients with an endpoint for retrieving all product details with a single request. It invokes various services, like product info and reviews, and combines the results*

Some months ago i discussed about the service mesh  
https://github.com/lorenzogirardi/kubernetes-servicemesh  

To better understand the differences on those products it is better to summarize the common areas

![apigw_vs_servicemesh](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_1024/v1604703880/misc/apigw_vs_servicemesh.png "apigw_vs_servicemesh")   

So forget about service mesh and have the focus only on the apigw.  

<br></br>

## GOAL 
<br></br>

Needed:
* Api must be public
* Users needs to have authentication
* An user could be rate limited
* A service could be rate limited
* Backend applications should not be changed
* We need to interact as a code with the apigw  
<br></br>  

Important:
* Analytics
* Transformation
* Versioning
* Circuit Breaker 
* gRPC support
* Caching
* Documentation

<br></br>  

## Software opportunity

Around the api gateway world we have some possibility, some of those are onprem, some other in cloud or hybrid.  

The onprem scenario we have some historycal software and a "new generation" born in a kubernetes architecture, howevere here we will check for a tool that could work in kubernetes (with a plus for supporting vm)

From the most knew i'd like to mention:
* Kong
* Tyk
* Nginx
* Ambassador
* 3scale

On the other side on the cloud scenario we have:
* Aws api gateway
* Mashery (from Tibco)
* Google apigee
* Akana (hybrid)
* Azure api management  
<br></br>  

Cloud  

![apigw_cloud](https://res.cloudinary.com/ethzero/image/upload/v1604766802/misc/apigw_cloud.png "apigw_cloud")   


The advantage/disadvantage of cloud solution is the pay per use, you don't need to manage the infrastrcture but only the configuration and you have the support that can drive you on the right configuration.  
Another valid point of view is the *DDOS* attack that will not land to your platform but to a cloud infrastructure that is supposed was designed to manage this scenario.

PRO:
- managed infrastructure 
- support
- attacks mitigation

CONS: 
- pricing 
- often not customizable
<br></br>

Onprem  

![apigw_dc](https://res.cloudinary.com/ethzero/image/upload/v1604766802/misc/apigw_dc.png "apigw_dc")  

Those solutions should be hosted on your platform or in a cloud to let the datacenter *unknown* (anyway you have to pay per traffic  "*" ).  
Your engineer team need to know how it's working the infrastructure but they can also improve the process with automations 

PRO:
- open source
- customizable
- awareness on the complete funnel  

CONS:
- you have to take care about attacks
- infrastructure knowhow


<br></br>
"*" you can easly have the apigw hosted onprem combined with a cdn/waf with ip lockdown , however you still have to pay the traffic towards it.  

![apigw_waf](https://res.cloudinary.com/ethzero/image/upload/v1604766802/misc/apigw_waf.png "apigw_waf")  

<br></br>

## Scenario

I tried some cloud solution like Aws and Mashery,  
i played also with some onprem solution, anyway i found out the right compromise with [Kong](https://konghq.com/kong/)

where compromise is :
- **Kubernetes ingress**
- **Api managed**
- **Documentation**
- Admin UI (konga for the open source)
- Basic auth
- **Apikey**
- **Oauth**
- **JWT**
- ip listing
- **Analytics**
- **Rate limiting**
- **Transformation**
- gRPC
- Websockets
- Service mesh (~ nice to have)  
- **Easy to use**



<br></br>

## Infrastructure

- kubernetes
- backend application
- kong api gateway
- konga admin ui
- waf (free cloudflare service)

<br></br>

What we need for this test is a backend api application, i created a prototype just to simulate different applications behind the apigw.

Here you can find out more details

https://github.com/lorenzogirardi/py-test-backend

briefly recap the application answer on /api/ with the main html page with methods  

| HTTP Method |                       URI                         | Action                     |
|-------------|:-------------------------------------------------:|----------------------------|
| GET         | http://[hostname]/api/get/context                 | Retrieve list of context   |
| GET         | http://[hostname]/api/get/context/[context_id]    | Retrieve a context         |
| POST        | http://[hostname]/api/post/context                | Create a new context       |
| PUT         | http://[hostname]/api/put/context/[context_id]    | Update an existing context |
| DELETE      | http://[hostname]/api/delete/context/[context_id] | Delete acontext            |   


<br></br>

## Configuration

Let's start to work...  
I've used the public kong deployment with some changes to have the database where store the configurations.

Another trick is create it as a secondary *ingress* , since i've already
and ingress in my ingrastructure , kong is done to answer on different ports

All the yaml are subdivided per role

```
kong
|-- 01-kong-ns.yaml
|-- 02-kong-cr.yaml
|-- 03-kong-sa.yaml
|-- 04-kong-rbac.yaml
|-- 05-kong-crb.yaml
|-- 06-kong-svc.yaml
|-- 07-kong-dpl.yaml
|-- 08-kong-ss.yaml
`-- 09-kong-job.yaml
```

some differences are related to the services node ports, deployment binding host and the database is now hosted with a volumeClaimTemplates  

```
  ports:
  - name: proxy
    nodePort: 30080
    port: 80
    protocol: TCP
    targetPort: 8000
  - name: proxy-ssl
    nodePort: 30443
    port: 443
    protocol: TCP
    targetPort: 8443
  - name: admin
    port: 8100
    protocol: TCP
    targetPort: 8100
  - name: admin-ssl
    port: 8444
    protocol: TCP
    targetPort: 8444
  selector:
    app: ingress-kong
  type: LoadBalancer
```

```
      containers:
      - env:
        - name: KONG_DATABASE
          value: postgres
        - name: KONG_PG_HOST
          value: postgres
        - name: KONG_PG_PASSWORD
          value: kong
        - name: KONG_PROXY_LISTEN
          value: 0.0.0.0:8000, 0.0.0.0:8443 ssl http2
        - name: KONG_PORT_MAPS
          value: 80:8000, 443:8443
        - name: KONG_ADMIN_LISTEN
          value: 0.0.0.0:8444 ssl
        - name: KONG_STATUS_LISTEN
          value: 0.0.0.0:8100
```

```
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 250Mi
```

Here what is expected to see

```
$ kubectl get svc -n kong
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                    AGE
kong-proxy                LoadBalancer   10.152.183.55    <pending>     80:30080/TCP,443:30443/TCP,8100:32588/TCP,8444:31793/TCP   7d
kong-validation-webhook   ClusterIP      10.152.183.190   <none>        443/TCP                                                    7d
postgres                  ClusterIP      10.152.183.177   <none>        5432/TCP
```

and the pods needed to have it online

```
kong           ingress-kong-686b4bc5df-nm7zk              2/2     Running     4          7d
kong           kong-migrations-ftgdv                      0/1     Completed   0          7d
kong           postgres-0                                 1/1     Running     2          7d
kube-system    hostpath-provisioner-5c65fbdb4f-f5qsb      1/1     Running     6          40d
```

```
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
datadir-postgres-0   Bound    pvc-8c5791c3-939f-4457-b5fb-74ac417ca3d7   250Mi        RWO                 k8s-hostpath   7d
```
<br></br>

Now i'd like to have also an Admin UI, i'm not fan of web ui ,  
however i need to know if this interface could be shared outside IT  
to delegate some business ownership and so on.

Kong Enterprise has it's own web ui instead the community one needs a third party tool... [Konga](https://github.com/pantsel/konga)  

In the same way of kong, konga needs a database , and in the same way i elaborated a bit the default deployment
```
konga
|-- 01-ns-konga.yaml
|-- 02-svc-konga.yaml
|-- 03-ing-konga.yaml
`-- 04-dpl-konga.yaml
```

just a couple of TIPS

postgres persistent storage  
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: konga
  name: konga-pv-claim
  labels:
    app: konga-storage-claim
spec:
  storageClassName: microk8s-hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Mi
```

and on deployment ...

```
        env:
        - name: NODE_TLS_REJECT_UNAUTHORIZED
          value: "0"
        - name: NODE_ENV
          value: "production"
```

where NODE_TLS_REJECT_UNAUTHORIZED is the option to not verify the kong api certificate and  
NODE_ENV is way to enable production or development , if you are not using production, the other two option enables the debug model and increase the cpu usage, i suggest to use production anyway **during the first deploy you should enable development in order to bootstrap the sql schema** that production is not able to do ... or create an init container to deploy the schema

```konga          konga-stable-8957b9d85-tq72z               2/2     Running     2          6d3h```

since is now up and running it will be available on the standard infrastructure ingress  

```
  - host: konga.ing.h4x0r3d.lan
```

Konga Home
![konga_home](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_1024/v1604777237/misc/konga_ui.png "konga_home")   

Konga node configuration
![konga_node](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_1024/v1604777238/misc/konga_ui_kongnode.png "konga_node")  

<br></br>

- kong [INSTALLED]
- konga [INSTALLED]
- backend [INSTALLED]  

First of all we need to expose the backend with Kong
in this example i created an ingress configuration dedicated for this porpose

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "kong"
  name: api
  namespace: pytbak
  labels:
    app: api
spec:
  rules:
    - host: api.ing.h4x0r3d.lan
      http:
        paths:
          - path: /api/
            backend:
              serviceName: pytbak-svc
              servicePort: 5000
          - path: /api/get/
            backend:
              serviceName: pytbak-svc
              servicePort: 5000
          - path: /api/post/
            backend:
              serviceName: pytbak-svc
              servicePort: 5000
          - path: /api/put/
            backend:
              serviceName: pytbak-svc
              servicePort: 5000
          - path: /api/delete/
            backend:
              serviceName: pytbak-svc
              servicePort: 5000
```

```kubernetes.io/ingress.class: "kong" ``` to match the kong ingress on the backend application
```namespace: pytbak```.  
As you can see i created mutiple paths to simulate different applications behind and create different ruls per path/application/context.  

The kong configuration could be applied with
- kubernetes yaml
- kong api
- konga ui (only if you have the database)

In the repo you will see some in kubernetes and some other not present , 
i tested the three option and the **kong api** is the best one  

<br></br>
Create api user

```curl -k -i -X POST --url https://192.168.1.14:31793/consumers/ --data "username=slow_user"```

```
HTTP/1.1 201 Created
Date: Sat, 07 Nov 2020 19:46:05 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/2.1.4
Content-Length: 121
X-Kong-Admin-Latency: 16
```

```{"custom_id":null,"created_at":1604778365,"id":"3d45a73d-4024-4813-b356-43a14ecea702","tags":null,"username":"slow_user"}```
![konga_consumer](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_1024/v1604778903/misc/konga_consumer.png "konga_consumer")

<br></br>
add a key token with key-auth plugin  

``` curl -k -i -X POST --url https://192.168.1.14:31793/consumers/slow_user/key-auth/ --data 'key=332d05445a560ee65a76aeaa372d8904' ```

```
HTTP/1.1 201 Created
Date: Sat, 07 Nov 2020 19:51:03 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/2.1.4
Content-Length: 190
X-Kong-Admin-Latency: 10
```

```{"created_at":1604778663,"id":"52970ca8-848d-49b5-a16f-a235fe0e9ceb","tags":null,"ttl":null,"key":"332d05445a560ee65a76aeaa372d8904","consumer":{"id":"3d45a73d-4024-4813-b356-43a14ecea702"}}```
![konga_consumer_cred](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_1024/v1604778903/misc/konga_consumer_credential.png "konga_consumer_cred")

<br></br>


routes.id where created when we created the kong ingress configuration for pytbak 
```curl -k -i -X GET https://192.168.1.14:31793/routes/```

name: pytbak.api.00 &nbsp; path:/api/  &nbsp; route.id: 0a78b572-cfe0-4228-bdfe-639ef04b5117    
name: pytbak.api.01 &nbsp; path:/api/get/  &nbsp; route.id: 8d2a24aa-636d-43ee-b0aa-b0fd1d3643fd  
name: pytbak.api.02 &nbsp; path:/api/post/  &nbsp; route.id: f75ed3b6-07c3-4293-932b-4f53d0c38f74    
name: pytbak.api.03 &nbsp; path:/api/put  &nbsp; route.id: f4823063-04e4-4da8-9c11-21977a93b1ba    
name: pytbak.api.04 &nbsp; path:/api/delete  &nbsp; route.id: e396750c-9158-4a96-8d79-047197669cff  


![konga_routes](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_1024/v1604779720/misc/konga_routes.png "konga_routes")

<br></br>
now i'd like to limit the user slow_user to perform only 4 request per sec on /api/  
and 2 on /api/get/

```
$ curl -k -X POST https://192.168.1.14:31793/plugins/ \
>             --data "name=rate-limiting"  \
>             --data "config.second=4" \
>             --data "config.hour=1200" \
>             --data "config.policy=local" \
>             --data "consumer.id=3d45a73d-4024-4813-b356-43a14ecea702" \
>             --data "route.id=0a78b572-cfe0-4228-bdfe-639ef04b5117"
```
```{"created_at":1604780031,"id":"cfd3d752-7864-493c-a3b7-3970ab6eca86","tags":null,"enabled":true,"protocols":["grpc","grpcs","http","https"],"name":"rate-limiting","consumer":{"id":"3d45a73d-4024-4813-b356-43a14ecea702"},"service":null,"route":{"id":"0a78b572-cfe0-4228-bdfe-639ef04b5117"},"config":{"hide_client_headers":false,"minute":null,"policy":"local","month":null,"redis_timeout":2000,"limit_by":"consumer","redis_password":null,"second":4,"day":null,"redis_database":0,"year":null,"hour":1200,"redis_host":null,"redis_port":6379,"header_name":null,"fault_tolerant":true}}```



```
curl -k -X POST https://192.168.1.14:31793/plugins/ \
                --data "name=rate-limiting"  \
                --data "config.second=2" \
                --data "config.hour=600" \
                --data "config.policy=local" \
                --data "consumer.id=3d45a73d-4024-4813-b356-43a14ecea702" \
                --data "route.id=8d2a24aa-636d-43ee-b0aa-b0fd1d3643fd" 
```
```{"created_at":1604780101,"id":"33bfc138-776a-4116-a2a9-8c70ab9d7a2a","tags":null,"enabled":true,"protocols":["grpc","grpcs","http","https"],"name":"rate-limiting","consumer":{"id":"3d45a73d-4024-4813-b356-43a14ecea702"},"service":null,"route":{"id":"8d2a24aa-636d-43ee-b0aa-b0fd1d3643fd"},"config":{"hide_client_headers":false,"minute":null,"policy":"local","month":null,"redis_timeout":2000,"limit_by":"consumer","redis_password":null,"second":2,"day":null,"redis_database":0,"year":null,"hour":600,"redis_host":null,"redis_port":6379,"header_name":null,"fault_tolerant":true}}```


![konga_rate_routes](https://res.cloudinary.com/ethzero/image/upload/c_scale,w_1024/v1604780200/misc/konga_rate_limit_routes.png "konga_rate_routes")


<br></br>

## Use cases

I created 3 users , one on the example above and a new one with a double of requests per second

| route        | slow_user rate/s | fast_user rate/s |
|--------------|:----------------:|------------------|
| /api/        | 4                | 8               |
| /api/get/    | 2                | 4                |
| /api/post/   | 1                | 2                |
| /api/put/    | 1                | 2                |
| /api/delete/ | 1                | 2                |  

<br></br>
slow_user has apikey: 332d05445a560ee65a76aeaa372d8904  
fast_user has apikey: 695aa0b18c6dbd1b387ee7c32c72c513   
admin has a apikey: admin  

<br></br>
i've also created a global rules on the service  with max 9 reqquest per second for all other users with no specific rate limit rule applied (ex admin.)  

```$ curl -k -X POST https://192.168.1.14:31793/plugins --data "name=rate-limiting" --data "config.second=9" --data "config.policy=local" --data "service.id=53d35e73-b285-4eb2-ad24-853d82563ca4" ```  

```
{"created_at":1604786501,"id":"34182311-ecbf-49a3-bdcf-bdc1c9c58706","tags":null,"enabled":true,"protocols":["grpc","grpcs","http","https"],"name":"rate-limiting","consumer":null,"service":{"id":"53d35e73-b285-4eb2-ad24-853d82563ca4"},"route":null,"config":{"hide_client_headers":false,"minute":null,"policy":"local","month":null,"redis_timeout":2000,"limit_by":"consumer","redis_password":null,"second":9,"day":null,"redis_database":0,"year":null,"hour":null,"redis_host":null,"redis_port":6379,"header_name":null,"fault_tolerant":true}}
```

Using siege i've benchmarked the users   


![siege_users](https://res.cloudinary.com/ethzero/image/upload/v1604787270/misc/siege_users.png "siege_users")

You can check the headers with curl  
```$ while true; do curl -I -H "apikey:332d05445a560ee65a76aeaa372d8904" http://api.ing.h4x0r3d.lan:30080/api/ && sleep 0.2; done```

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 658
Connection: keep-alive
Server: Werkzeug/1.0.1 Python/3.9.0
Date: Sat, 07 Nov 2020 22:21:34 GMT
X-RateLimit-Limit-Second: 4
X-RateLimit-Remaining-Second: 3
X-RateLimit-Limit-Hour: 1000000
RateLimit-Limit: 4
X-RateLimit-Remaining-Hour: 999755
RateLimit-Remaining: 3
RateLimit-Reset: 1
X-Kong-Upstream-Latency: 3
X-Kong-Proxy-Latency: 1
Via: kong/2.1.4

HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 658
Connection: keep-alive
Server: Werkzeug/1.0.1 Python/3.9.0
Date: Sat, 07 Nov 2020 22:21:34 GMT
X-RateLimit-Limit-Second: 4
X-RateLimit-Remaining-Second: 2
X-RateLimit-Limit-Hour: 1000000
RateLimit-Limit: 4
X-RateLimit-Remaining-Hour: 999754
RateLimit-Remaining: 2
RateLimit-Reset: 1
X-Kong-Upstream-Latency: 3
X-Kong-Proxy-Latency: 0
Via: kong/2.1.4

HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 658
Connection: keep-alive
Server: Werkzeug/1.0.1 Python/3.9.0
Date: Sat, 07 Nov 2020 22:21:34 GMT
X-RateLimit-Limit-Second: 4
X-RateLimit-Remaining-Second: 1
X-RateLimit-Limit-Hour: 1000000
RateLimit-Limit: 4
X-RateLimit-Remaining-Hour: 999753
RateLimit-Remaining: 1
RateLimit-Reset: 1
X-Kong-Upstream-Latency: 3
X-Kong-Proxy-Latency: 1
Via: kong/2.1.4

HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 658
Connection: keep-alive
Server: Werkzeug/1.0.1 Python/3.9.0
Date: Sat, 07 Nov 2020 22:21:34 GMT
X-RateLimit-Limit-Second: 4
X-RateLimit-Remaining-Second: 0
X-RateLimit-Limit-Hour: 1000000
RateLimit-Limit: 4
X-RateLimit-Remaining-Hour: 999752
RateLimit-Remaining: 0
RateLimit-Reset: 1
X-Kong-Upstream-Latency: 3
X-Kong-Proxy-Latency: 1
Via: kong/2.1.4

HTTP/1.1 429 Too Many Requests
Date: Sat, 07 Nov 2020 22:21:34 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Retry-After: 1
Content-Length: 41
X-RateLimit-Limit-Second: 4
X-RateLimit-Remaining-Second: 0
X-RateLimit-Limit-Hour: 1000000
RateLimit-Limit: 4
X-RateLimit-Remaining-Hour: 999752
RateLimit-Remaining: 0
RateLimit-Reset: 1
X-Kong-Response-Latency: 1
Server: kong/2.1.4
```
<br></br>

### Rules Precedence
A plugin will always be run once and only once per request. But the configuration with which it will run depends on the entities it has been configured for.  

Plugins can be configured for various entities, combination of entities, or even globally. This is useful, for example, when you wish to configure a plugin a certain way for most requests, but make authenticated requests behave slightly differently.  

Therefore, there exists an order of precedence for running a plugin when it has been applied to different entities with different configurations. The rule of thumb is: the more specific a plugin is with regards to how many entities it has been configured on, the higher its priority.  

The complete order of precedence when a plugin has been configured multiple times is:

1. Plugins configured on a combination of: a Route, a Service, and a Consumer. (Consumer means the request must be authenticated).
2. Plugins configured on a combination of a Route and a Consumer. (Consumer means the request must be authenticated).
3. Plugins configured on a combination of a Service and a Consumer. (Consumer means the request must be authenticated).
4. Plugins configured on a combination of a Route and a Service.
5. Plugins configured on a Consumer. (Consumer means the request must be authenticated).
6. Plugins configured on a Route.
7. Plugins configured on a Service.
8. Plugins configured to run globally.  

Example: if the rate-limiting plugin is applied twice (with different configurations): for a Service (Plugin config A), and for a Consumer (Plugin config B), then requests authenticating this Consumer will run Plugin config B and ignore A. However, requests that do not authenticate this Consumer will fallback to running Plugin config A. Note that if config B is disabled (its enabled flag is set to false), config A will apply to requests that would have otherwise matched config B.

<br></br>

### WAF

Last but not lease , the api is configured behind Cloudflare,    

the users ```slow_user fast_user``` are available in case you want to test directly the service.  

Endpoint: ```https://services.k8s.it/api/```

<br></br>

```$ curl -I -H "apikey:332d05445a560ee65a76aeaa372d8904" https://services.k8s.it/api/```
```
HTTP/2 200
date: Sun, 08 Nov 2020 10:14:21 GMT
content-type: text/html; charset=utf-8
set-cookie: __cfduid=dab5b88be5427314b7b17a1a65d3178a41604830461; expires=Tue, 08-Dec-20 10:14:21 GMT; path=/; domain=.k8s.it; HttpOnly; SameSite=Lax
vary: Accept-Encoding
x-ratelimit-limit-second: 4
x-ratelimit-remaining-second: 3
x-ratelimit-limit-hour: 1000000
ratelimit-limit: 4
x-ratelimit-remaining-hour: 999995
ratelimit-remaining: 3
ratelimit-reset: 1
x-kong-upstream-latency: 3
x-kong-proxy-latency: 4
via: kong/2.1.4
cf-cache-status: DYNAMIC
cf-request-id: 0648f2980000000f72931c2000000001
expect-ct: max-age=604800, report-uri="https://report-uri.cloudflare.com/cdn-cgi/beacon/expect-ct"
report-to: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report?s=z4j%2FU3og%2FT2jBByhUKZtShDNLfLMWncExIy3EowJ2hOPJ17oCB0oeyY1GVaXirT%2BgWpTEEuU%2F03gx%2F91uOy3mGD%2BFE4FhBGqFt7iC2bhaPc%3D"}],"group":"cf-nel","max_age":604800}
nel: {"report_to":"cf-nel","max_age":604800}
server: cloudflare
cf-ray: 5eee86d319770f72-MXP
```



<br></br>

## Conclusion