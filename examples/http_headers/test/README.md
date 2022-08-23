## Refer to
- https://sirishagopigiri-692.medium.com/deploying-envoy-filter-on-istio-ce2d2573b981

## Prepare the wasm
```
# Building WASM binary
$ git clone https://github.com/tetratelabs/proxy-wasm-go-sdk.git
$ cd proxy-wasm-go-sdk
$ cd examples/http_headers
$ tinygo build -o ./main.go.wasm -scheduler=none -target=wasi ./main.go
$ ls
README.md  envoy.yaml  main.go  main.go.wasm  main_test.go
```

## Configure

```
# Deploy
kubectl apply -f https://raw.githubusercontent.com/SirishaGopigiri/Istio-WASM-plugin/main/httpbin.yaml
##################################################################################################
# httpbin service
##################################################################################################
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
    service: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
      annotations:
        sidecar.istio.io/userVolume: '[{"name":"http-filter","configMap":{"name":"http-filter"}}]'
        sidecar.istio.io/userVolumeMount: '[{"mountPath":"/var/local/wasm","name":"http-filter"}]'
        sidecar.istio.io/logLevel: "info"
    spec:
      serviceAccountName: httpbin
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80

# Filter
kubectl apply -f https://raw.githubusercontent.com/SirishaGopigiri/Istio-WASM-plugin/main/filter.yaml

apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: golang-filter
spec:
  workloadSelector:
    labels:
      app: httpbin
  configPatches:
    # The first patch adds the lua filter to the listener/http connection manager
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
            subFilter:
              name: envoy.router
    patch:
      operation: INSERT_BEFORE
      value: # lua filter specification
        name: envoy.filters.http.wasm
        typed_config:
          '@type': type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
          value:
            config:
              vm_config:
                code:
                  local:
                    filename: /var/local/wasm/http-filter.wasm
                runtime: envoy.wasm.runtime.v8
```

## Test
```
ubuntu@ip-172-31-41-18:~$ k exec -ti nginx -- curl httpbin:8000/status/200 -v
*   Trying 10.100.202.128...
* TCP_NODELAY set
* Connected to httpbin (10.100.202.128) port 8000 (#0)
> GET /status/200 HTTP/1.1
> Host: httpbin:8000
> User-Agent: curl/7.52.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< server: envoy
< date: Tue, 23 Aug 2022 04:08:42 GMT
< content-type: text/html; charset=utf-8
< access-control-allow-origin: *
< access-control-allow-credentials: true
< content-length: 0
< x-envoy-upstream-service-time: 18
< 
* Curl_http_done: called premature == 0
* Connection #0 to host httpbin left intact

ubuntu@ip-172-31-41-18:~$ k logs -n default deploy/httpbin  -c istio-proxy --tail 30 -f
2022-08-23T03:47:18.626672Z	info	envoy upstream	cds: add 31 cluster(s), remove 5 cluster(s)
2022-08-23T03:47:18.629439Z	info	envoy upstream	cds: added/updated 0 cluster(s), skipped 31 unmodified cluster(s)
2022-08-23T04:08:42.833682Z	info	envoy wasm	wasm log: request header --> :authority: httpbin:8000
2022-08-23T04:08:42.833804Z	info	envoy wasm	wasm log: request header --> :path: /status/200
2022-08-23T04:08:42.833827Z	info	envoy wasm	wasm log: request header --> :method: GET
2022-08-23T04:08:42.833881Z	info	envoy wasm	wasm log: request header --> :scheme: http
2022-08-23T04:08:42.833910Z	info	envoy wasm	wasm log: request header --> user-agent: curl/7.52.1
2022-08-23T04:08:42.833974Z	info	envoy wasm	wasm log: request header --> accept: */*
2022-08-23T04:08:42.834045Z	info	envoy wasm	wasm log: request header --> x-forwarded-proto: http
2022-08-23T04:08:42.834090Z	info	envoy wasm	wasm log: request header --> x-request-id: 20a2c568-5095-47eb-a028-66939f5a3ced
2022-08-23T04:08:42.834125Z	info	envoy wasm	wasm log: request header --> x-envoy-attempt-count: 1
2022-08-23T04:08:42.834171Z	info	envoy wasm	wasm log: request header --> x-b3-traceid: 6347ef3361aa1c020276a1379b420e8a
2022-08-23T04:08:42.834203Z	info	envoy wasm	wasm log: request header --> x-b3-spanid: 0276a1379b420e8a
2022-08-23T04:08:42.834253Z	info	envoy wasm	wasm log: request header --> x-b3-sampled: 0
2022-08-23T04:08:42.834287Z	info	envoy wasm	wasm log: request header --> x-forwarded-client-cert: By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=57515f43a52ff4eed1161ecb22b9da91239f0c7febf5ba498770e4b5de55f15e;Subject="";URI=spiffe://cluster.local/ns/default/sa/default
2022-08-23T04:08:42.834333Z	info	envoy wasm	wasm log: request header --> test: best
2022-08-23T04:08:42.836393Z	info	envoy wasm	wasm log: response header <-- :status: 200
2022-08-23T04:08:42.836502Z	info	envoy wasm	wasm log: response header <-- server: gunicorn/19.9.0
2022-08-23T04:08:42.836523Z	info	envoy wasm	wasm log: response header <-- date: Tue, 23 Aug 2022 04:08:42 GMT
2022-08-23T04:08:42.836565Z	info	envoy wasm	wasm log: response header <-- connection: keep-alive
2022-08-23T04:08:42.836603Z	info	envoy wasm	wasm log: response header <-- content-type: text/html; charset=utf-8
2022-08-23T04:08:42.836649Z	info	envoy wasm	wasm log: response header <-- access-control-allow-origin: *
2022-08-23T04:08:42.836661Z	info	envoy wasm	wasm log: response header <-- access-control-allow-credentials: true
2022-08-23T04:08:42.836665Z	info	envoy wasm	wasm log: response header <-- content-length: 0
2022-08-23T04:08:42.836668Z	info	envoy wasm	wasm log: response header <-- x-envoy-upstream-service-time: 1
2022-08-23T04:08:42.844561Z	info	envoy wasm	wasm log: 2 finished
[2022-08-23T04:08:42.833Z] "GET /status/200 HTTP/1.1" 200 - via_upstream - "-" 0 0 3 1 "-" "curl/7.52.1" "20a2c568-5095-47eb-a028-66939f5a3ced" "httpbin:8000" "192.168.86.198:80" inbound|80|| 127.0.0.6:44019 192.168.86.198:80 192.168.63.207:50766 outbound_.8000_._.httpbin.default.svc.cluster.local default

```
