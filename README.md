# Ambassador GH-685 Wtfsauce

## Scenario!

As an end user I want to deploy Ambassador and the following are requirements:

1. I am deploying on AWS
1. I MUST terminate SSL/TLS at my Elastic Load Balancer.
2. I MUST support both HTTP (:80) and HTTPS (:443) traffic, further, The HTTP (:80) traffic MUST BE redirected to HTTPS (:443).
3. I MUST support websocket traffic (and therefore MUST use a L4 listener on the ELB)
4. I MUST receive the **REAL** client IP address at my backend (and there MUST use the Proxy Protocol)

## Basic Ambassador Config

1. `service.beta.kubernetes.io/aws-load-balancer-backend-protocol` is set to TCP to ensure L4 mode.
2. `service.beta.kubernetes.io/aws-load-balancer-ssl-cert` references an SSL/TLS cert in AWS Certificate Manager.
3. `service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"` ensures that SSL/TLS is ONLY enabled on ELB port 443.
4. `service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"` ensures that Proxy Protocol is enabled.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: ambassador-public
  namespace: gh685
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:914373874199:certificate/1d16f0a1-abd5-44b3-8da5-5e06c843fd3d"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v0
      kind: Module
      name: ambassador
      config:
        use_proxy_proto: true    # ensures Proxy Protocol information is read by Envoy.
        use_remote_address: true # ensures Envoy sets `X-Forwarded-For` with source IP from Proxy Protocol.

      ---
      apiVersion: ambassador/v1
      kind: Module
      name: tls
      config:
        server:
          enabled: false              
          redirect_cleartext_from: 80
        client:
          enabled: false
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80       # configures an ELB port as :80
    targetPort: 80
  - name: https
    port: 443      # configures an ELB listener on :443 (and because of an annotation it will be an SSL listener)
    targetPort: 80
  selector:
    app: ambassador
```

## Setup

1. Get an AWS Kubernetes cluster
2. `kubectl create ns gh685`
3. `kubectl apply -f ambassador.core.yaml` (this is all boring stuff: RBAC, Ambassador and backend deployment)
4. `kubectl apply -f ambassador.svc.yaml`  (this is where our configuration lives)

# Issue #1

Copy and paste this into `ambassador.svc.yaml` and then `kubectl apply -f ambassador.svc.yaml`. This **SHOULD BE** the canonical way to do this on AWS. In the `HTTP/:80` case I was expecting a 301 or 302 redirect.

    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: ambassador-public
    namespace: gh685
    annotations:
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
        service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
        service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:914373874199:certificate/1d16f0a1-abd5-44b3-8da5-5e06c843fd3d"
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
        service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
        getambassador.io/config: |
        ---
        apiVersion: ambassador/v0
        kind: Module
        name: ambassador
        config:
            use_proxy_proto: true
            use_remote_address: true

        ---
        apiVersion: ambassador/v1
        kind: Module
        name: tls
        config:
            server:
                enabled: false
                redirect_cleartext_from: 80
            client:
                enabled: false
    spec:
    type: LoadBalancer
    ports:
    - name: http
        port: 80
        targetPort: 80
    - name: https
        port: 443
        targetPort: 80
    selector:
        app: ambassador
    ```

1. `export LB_ADDR=$(kubectl get svc ambassador-public -n gh685 -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}')`
2. `curl -k -v https://$LB_ADDR`

    **GOOD:** You should get a TLS handshake and that's great:
    
    ```text
    curl -k -v https://$LB_ADDR
    * Rebuilt URL to: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    *   Trying 34.234.177.126...
    * TCP_NODELAY set
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (34.234.177.126) port 443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * Cipher selection: PROFILE=SYSTEM
    * successfully set certificate verify locations:
    *   CAfile: /etc/pki/tls/certs/ca-bundle.crt
    CApath: none
    * TLSv1.2 (OUT), TLS handshake, Client hello (1):
    * TLSv1.2 (IN), TLS handshake, Server hello (2):
    * TLSv1.2 (IN), TLS handshake, Certificate (11):
    * TLSv1.2 (IN), TLS handshake, Server key exchange (12):
    * TLSv1.2 (IN), TLS handshake, Server finished (14):
    * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
    * TLSv1.2 (OUT), TLS change cipher, Client hello (1):
    * TLSv1.2 (OUT), TLS handshake, Finished (20):
    * TLSv1.2 (IN), TLS handshake, Finished (20):
    * SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
    * ALPN, server did not agree to a protocol
    * Server certificate:
    *  subject: CN=*.getambassador.io
    *  start date: May 29 00:00:00 2018 GMT
    *  expire date: Jun 29 12:00:00 2019 GMT
    *  issuer: C=US; O=Amazon; OU=Server CA 1B; CN=Amazon
    *  SSL certificate verify ok.
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 200 OK
    < date: Tue, 31 Jul 2018 21:45:15 GMT
    < server: envoy
    < content-type: application/json;charset=utf-8
    < content-length: 764
    < x-envoy-upstream-service-time: 6
    < 
    {
    "host" : "backend-6f77f444c4-2pn79",
    "request" : {
        "method" : "GET",
        "path" : "/",
        "queryParams" : {
        "" : [ "" ]
        },
        "headers" : {
        "x-request-id" : "3bf7271e-d8f3-4640-ad41-5fec6034d5c3",
        "x-envoy-external-address" : "65.217.185.138",
        "Accept" : "*/*",
        "User-Agent" : "curl/7.55.1",
        "X-Forwarded-Proto" : "http",
        "X-Forwarded-For" : "65.217.185.138",
        "Host" : "aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com",
        "x-envoy-expected-rq-timeout-ms" : "3000",
        "Content-Length" : "0",
        "x-envoy-original-path" : "/"
        },
        "remoteAddress" : "100.105.109.197",
        "remotePort" : 60630,
        "remoteHost" : "100.105.109.197",
        "remoteUser" : null
    }
    * Connection #0 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    }
    ```

3. `curl -v http://$LB_ADDR`

    **BAD:** No TLS handshake occurred
    
    ```text
    curl -k -v http://$LB_ADDR
    * Rebuilt URL to: http://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    *   Trying 52.0.246.212...
    * TCP_NODELAY set
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (52.0.246.212) port 80 (#0)
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 200 OK
    < date: Tue, 31 Jul 2018 21:46:37 GMT
    < server: envoy
    < content-type: application/json;charset=utf-8
    < content-length: 764
    < x-envoy-upstream-service-time: 4
    < 
    {
    "host" : "backend-6f77f444c4-2pn79",
    "request" : {
        "method" : "GET",
        "path" : "/",
        "queryParams" : {
        "" : [ "" ]
        },
        "headers" : {
        "x-request-id" : "100d49ee-21ce-4973-b481-a634dff93838",
        "x-envoy-external-address" : "104.129.66.226",
        "Accept" : "*/*",
        "User-Agent" : "curl/7.55.1",
        "X-Forwarded-Proto" : "http",
        "X-Forwarded-For" : "104.129.66.226",
        "Host" : "aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com",
        "x-envoy-expected-rq-timeout-ms" : "3000",
        "Content-Length" : "0",
        "x-envoy-original-path" : "/"
        },
        "remoteAddress" : "100.105.109.197",
        "remotePort" : 60834,
        "remoteHost" : "100.105.109.197",
        "remoteUser" : null
    }
    * Connection #0 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    }
    ```

# Issue #2 Infinite Redirect

Copy and paste this into `ambassador.svc.yaml` and then `kubectl apply -f ambassador.svc.yaml`. The difference here between the first issue (above) is that `config.server.enabled` is now `true` which is nonsensical for two reasons:

1. The docs say `enabled: true` is only for TLS termination. We are not terminating TLS on Ambassador.
2. The docs, however, do not tell you to set `enabled: false` if you are terminating TLS at your load balancer.
3. Therefore... what's a user to do other than try?!

    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: ambassador-public
    namespace: gh685
    annotations:
        service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
        service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
        service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:914373874199:certificate/1d16f0a1-abd5-44b3-8da5-5e06c843fd3d"
        service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
        service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
        getambassador.io/config: |
        ---
        apiVersion: ambassador/v0
        kind: Module
        name: ambassador
        config:
            use_proxy_proto: true
            use_remote_address: true

        ---
        apiVersion: ambassador/v1
        kind: Module
        name: tls
        config:
            server:
                enabled: true
                redirect_cleartext_from: 80
            client:
                enabled: false
    spec:
    type: LoadBalancer
    ports:
    - name: http
        port: 80
        targetPort: 80
    - name: https
        port: 443
        targetPort: 80
    selector:
        app: ambassador
    ```
    
1. `export LB_ADDR=$(kubectl get svc ambassador-public -n gh685 -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}')`
2. `curl -k -v https://$LB_ADDR`

    ```text
    * Rebuilt URL to: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    *   Trying 52.0.246.212...
    * TCP_NODELAY set
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (52.0.246.212) port 443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * Cipher selection: PROFILE=SYSTEM
    * successfully set certificate verify locations:
    *   CAfile: /etc/pki/tls/certs/ca-bundle.crt
    CApath: none
    * TLSv1.2 (OUT), TLS handshake, Client hello (1):
    * TLSv1.2 (IN), TLS handshake, Server hello (2):
    * TLSv1.2 (IN), TLS handshake, Certificate (11):
    * TLSv1.2 (IN), TLS handshake, Server key exchange (12):
    * TLSv1.2 (IN), TLS handshake, Server finished (14):
    * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
    * TLSv1.2 (OUT), TLS change cipher, Client hello (1):
    * TLSv1.2 (OUT), TLS handshake, Finished (20):
    * TLSv1.2 (IN), TLS handshake, Finished (20):
    * SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
    * ALPN, server did not agree to a protocol
    * Server certificate:
    *  subject: CN=*.getambassador.io
    *  start date: May 29 00:00:00 2018 GMT
    *  expire date: Jun 29 12:00:00 2019 GMT
    *  issuer: C=US; O=Amazon; OU=Server CA 1B; CN=Amazon
    *  SSL certificate verify ok.
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 301 Moved Permanently
    < location: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    < date: Tue, 31 Jul 2018 21:57:42 GMT
    < server: envoy
    < content-length: 0
    < 
    * Connection #0 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    ```
    
    Yuck... we are not redirecting on HTTPS. That's going to be a problem for an HTTP:80 redirect.
    
3. `curl -k -v -L https://$LB_ADDR`

    If you are following along at home... the `-L` is to follow redirects and if you use it YOU WILL end up in an infinite redirect loop as HTTP redirects to HTTPS and HTTPS redirects to itself. In the below example I set `--max-redirs` for sanity reasons.
    
    ```
    plombardi@plombowski ~/w/r/a/gh685> curl -k -v -L --max-redirs 5 http://$LB_ADDR
    * Rebuilt URL to: http://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    *   Trying 52.0.246.212...
    * TCP_NODELAY set
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (52.0.246.212) port 80 (#0)
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 301 Moved Permanently
    < location: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    < date: Tue, 31 Jul 2018 22:01:15 GMT
    < server: envoy
    < content-length: 0
    < 
    * Connection #0 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    * Issue another request to this URL: 'https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/'
    *   Trying 34.194.124.191...
    * TCP_NODELAY set
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (34.194.124.191) port 443 (#1)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * Cipher selection: PROFILE=SYSTEM
    * successfully set certificate verify locations:
    *   CAfile: /etc/pki/tls/certs/ca-bundle.crt
    CApath: none
    * TLSv1.2 (OUT), TLS handshake, Client hello (1):
    * TLSv1.2 (IN), TLS handshake, Server hello (2):
    * TLSv1.2 (IN), TLS handshake, Certificate (11):
    * TLSv1.2 (IN), TLS handshake, Server key exchange (12):
    * TLSv1.2 (IN), TLS handshake, Server finished (14):
    * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
    * TLSv1.2 (OUT), TLS change cipher, Client hello (1):
    * TLSv1.2 (OUT), TLS handshake, Finished (20):
    * TLSv1.2 (IN), TLS handshake, Finished (20):
    * SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
    * ALPN, server did not agree to a protocol
    * Server certificate:
    *  subject: CN=*.getambassador.io
    *  start date: May 29 00:00:00 2018 GMT
    *  expire date: Jun 29 12:00:00 2019 GMT
    *  issuer: C=US; O=Amazon; OU=Server CA 1B; CN=Amazon
    *  SSL certificate verify ok.
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 301 Moved Permanently
    < location: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    < date: Tue, 31 Jul 2018 22:01:16 GMT
    < server: envoy
    < content-length: 0
    < 
    * Connection #1 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    * Issue another request to this URL: 'https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/'
    * Found bundle for host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com: 0x5649ec9b7100 [can pipeline]
    * Re-using existing connection! (#1) with host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (34.194.124.191) port 443 (#1)
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 301 Moved Permanently
    < location: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    < date: Tue, 31 Jul 2018 22:01:16 GMT
    < server: envoy
    < content-length: 0
    < 
    * Connection #1 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    * Issue another request to this URL: 'https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/'
    * Found bundle for host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com: 0x5649ec9b7100 [can pipeline]
    * Re-using existing connection! (#1) with host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (34.194.124.191) port 443 (#1)
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 301 Moved Permanently
    < location: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    < date: Tue, 31 Jul 2018 22:01:16 GMT
    < server: envoy
    < content-length: 0
    < 
    * Connection #1 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    * Issue another request to this URL: 'https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/'
    * Found bundle for host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com: 0x5649ec9b7100 [can pipeline]
    * Re-using existing connection! (#1) with host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (34.194.124.191) port 443 (#1)
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 301 Moved Permanently
    < location: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    < date: Tue, 31 Jul 2018 22:01:16 GMT
    < server: envoy
    < content-length: 0
    < 
    * Connection #1 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    * Issue another request to this URL: 'https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/'
    * Found bundle for host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com: 0x5649ec9b7100 [can pipeline]
    * Re-using existing connection! (#1) with host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    * Connected to aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com (34.194.124.191) port 443 (#1)
    > GET / HTTP/1.1
    > Host: aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com
    > User-Agent: curl/7.55.1
    > Accept: */*
    > 
    < HTTP/1.1 301 Moved Permanently
    < location: https://aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com/
    < date: Tue, 31 Jul 2018 22:01:16 GMT
    < server: envoy
    < content-length: 0
    < 
    * Connection #1 to host aaf17f53994ff11e884fb0e884ad5c65-1982898853.us-east-1.elb.amazonaws.com left intact
    * Maximum (5) redirects followed
    curl: (47) Maximum (5) redirects followed
    ```
    
# Closing Thoughts!

- The Ambassador docs around AWS ELB's are confusing (https://www.getambassador.io/reference/ambassador-with-aws). As far as I am aware the below is impossible with Ambassador today. There is no way to configure multiple listeners which is essentially how the below documentation would work. I think this is perhaps some internal note that made it into docs. If this were possible you would point `ELB:443 @ Envoy-Listener:8443` and `ELB:80 @ Envoy-Listener:8080` then we would configure `Envoy-Listener:8080` to redirect all routes to `ELB:443`... But I am not sure that is even possible so I don't know why it is in the documentation.

    ```text
    Run the ELB with the following listener configuration:

    :443 -> :8443 (the Envoy port doesn't matter)
    :80 -> :8080 (the Envoy port doesn't matter)
    Configure redirect_cleartext_from to redirect traffic on 8080 to the secure port
    ```

- The example shown for the AWS ELB config is suitable for TLS-only terminated setups because it only points `:443` backwards. We should update this with the appropriate configs AFTER we fix the existing issue.
- Envoy v2 API seems to have a more powerful redirect mechanism: https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/route/route.proto#route-redirectaction



