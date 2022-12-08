# Demo: Authorino

Demo of [Authorino](https://github.com/kuadrant/authorino), K8s-native external authorization service, to secure an application, the **News Agency API**, on Kubernetes, prepared for the [DevStaff AuthN/AuthZ Meetup](https://www.meetup.com/devstaff/events/289951423/) in Crete (Greece), 2022.

<br/>

The News Agency API ("News API" for short) is a REST API to manage news articles (Create, Read, Delete), with no embedded concept of authentication or authorization. Records are stored in a Redis database deployed together with the application.

HTTP endpoints available:
```
POST /{category}          Create a news article
GET /{category}           List news articles
GET /{category}/{id}      Read a news article
DELETE /{category}/{id}   Delete a news article
```

A news article is structured as follows:

```jsonc
{
  "id": <string: auto-generated>,
  "title": <string>,
  "body": <string>,
  "date": <string: ISO 8601>,
  "author": <string>,
  "user_id": <string>
}
```

The attributes `author` and `user_id` can only be supplied in a stringified JSON passed in the request in the `x-ext-auth-data` HTTP header.

The demo covers securing the News API with authentication based on API keys as well as Kubernetes Service Account tokens, and authorization based on Kubernetes SubjectAccessReview.

### Outline

![Outline](http://www.plantuml.com/plantuml/png/XP31IWCn48RlUOgXNXGHzEP9kbOFuaLiZu8CoMoxR7OI9XFelhrnbuLOAruc97nV_ZzP9qNHF7YJ-euZ2WxWgCNiTKT7RNotvu5OmPP1Kb4IChjD42Q1krjZXAmYxpt1wecY3oFeWG1Zz9r5xG9_y2K7mAo7gnLW0ZTHjRUJCrBcA44BH6xILCRFwWmkshQjBzcIpK9dmfkt5-XfX6julK_m_jXivXvf4lxjyRl5tnsUZqhi0Asbb4hqT-2s0GqzSPfJQKACcNy1RXvE7sPEzWLPgixBulmqQdu9MPUH1_y5)

### Stack

![Stack](http://www.plantuml.com/plantuml/png/SoWkIImgAStDuIfAJIv9p4lFILLGyYvDIYtAIor9BLPII2nMo8Pp5Qgv51IG5BhcbULNWjMaWbXWQHG5VgdbnGgE0PvWDNb0JdnYGIPGKIsgEOwb9HdvHPbv-M1rYJ0ULoqN5voZe0knXCiXDIy5w600)

### Prerequisites

- [Docker](https://www.docker.com/)
- [Kind](https://kind.sigs.k8s.io/)

## Deploy the application

**API lifecycle**

![Lifecycle](http://www.plantuml.com/plantuml/png/hP9DI_j04CRl-HH3Jl_ofvY-U2kbfIY8e89w2yWccIHBDxDbTbQidzvjWkG5RvfSChiFy_pcoUoSA1RVcCWTDPqKgmOAB9Ktye8ViZUweWP984SIv86AhQVYO9cGOP544MCkYYg346-oxM9pbMrJsZ_TWNRWwSHMWW2BbBft7fwKbaa2Z_Sng95cqcivwfLXhIcqkQ5tU_w_zr9RrcHJ-jTevpHL4CUNquEbKbTnFElTriaQ7gp0xGMzDIKhRsMeffQhpl8PSyzQpk3o6Xk4qZ98ZH1GKdAY0Yje0aKJpm3Z7JAKIXi7Oa65IoJHkH8S0ItWbLGtmoTsJ9w6wYdP-jTa1YijqF8PbHyTd93Rw2oDq5OX9yvqKI3rN1te5EhwR-8QpHra1VIEin-NnXwZQB0uCDyEVkdtLtiyJNLI1ybumBxeBeFJ3gdmZVa2)

<details>
  <summary><b>① Create the cluster</b></summary>

  ```sh
  kind create cluster --name demo
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kind%20create%20cluster%20--name%20demo)</sub>
</details>

<details>
  <summary><b>② Create the namespace</b></summary>

  ```sh
  kubectl create namespace news-agency
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20create%20namespace%20news-agency)</sub>
</details>

<details>
  <summary><b>③ Deploy the application</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: news-api
    labels:
      app: news-api
  spec:
    selector:
      matchLabels:
        app: news-api
    template:
      metadata:
        labels:
          app: news-api
      spec:
        containers:
        - name: news-api
          image: quay.io/kuadrant/authorino-examples:news-api
          imagePullPolicy: IfNotPresent
          env:
          - name: PORT
            value: "3000"
          - name: REDIS_URL
            value: redis://redis.news-agency.svc.cluster.local:6379
    replicas: 1
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: news-api
    labels:
      app: news-api
  spec:
    selector:
      app: news-api
    ports:
    - name: http
      port: 3000
      protocol: TCP
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: redis
    labels:
      app: redis
  spec:
    selector:
      matchLabels:
        app: redis
    template:
      metadata:
        labels:
          app: redis
      spec:
        containers:
        - name: redis
          image: redis
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 6379
    replicas: 1
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: redis
    labels:
      app: redis
  spec:
    selector:
      app: redis
    ports:
    - name: redis
      port: 6379
      protocol: TCP
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20news-api.yaml)</sub>
</details>

## Try the application unprotected

<details>
  <summary><b>① Expose the application</b></summary>

  ```sh
  kubectl port-forward -n news-agency service/news-api 3000:3000 2>&1 >/dev/null &;PID=$!
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20port-forward%20-n%20news-agency%20service/news-api%203000:3000%202%3E%261%20%3E/dev/null%20%26;PID=$!)</sub>
</details>

<details>
  <summary><b>② Send requests to the application unproctected</b></summary>

  ```sh
  curl http://api.news-agency.127.0.0.1.nip.io:3000/tech -s -i
  # HTTP/1.1 200 OK
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20http://api.news-agency.127.0.0.1.nip.io:3000/tech%20-s%20-i)</sub>

  ```sh
  curl -X POST \
       -d '{"title":"Risky online behaviour ‘almost normalised’ among young people, says study","body":"EU-funded survey of people aged 16-19 finds one in four have trolled someone – while UK least ‘cyberdeviant’ of nine countries (By Dan Milmo - The Guardian)"}' \
       http://api.news-agency.127.0.0.1.nip.io:3000/tech -s -i
  # HTTP/1.1 200 OK
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-X%20POST%20-d%20'%7B%22title%22:%22Risky%20online%20behaviour%20%E2%80%98almost%20normalised%E2%80%99%20among%20young%20people,%20says%20study%22,%22body%22:%22EU-funded%20survey%20of%20people%20aged%2016-19%20finds%20one%20in%20four%20have%20trolled%20someone%20%E2%80%93%20while%20UK%20least%20%E2%80%98cyberdeviant%E2%80%99%20of%20nine%20countries%20(By%20Dan%20Milmo%20-%20The%20Guardian)%22%7D'%20http://api.news-agency.127.0.0.1.nip.io:3000/tech%20-s%20-i)</sub>
</details>

## Secure the application

Secure the application with authentication based on API keys and authorization using the Kubernetes SubjectAccessReview (Kubernetes RBAC).

<details>
  <summary><b>① Install Authorino</b></summary>

  Install the Authorino Operator:

  ```sh
  kubectl apply -f https://raw.githubusercontent.com/Kuadrant/authorino-operator/main/config/deploy/manifests.yaml
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-f%20authorino-operator.yaml)</sub>

  Request an Authorino instance:

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: operator.authorino.kuadrant.io/v1beta1
  kind: Authorino
  metadata:
    name: authorino
  spec:
    listener:
      tls:
        enabled: false
    oidcServer:
      tls:
        enabled: false
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20authorino.yaml)</sub>
</details>

<details>
  <summary><b>② Inject the Envoy sidecar</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: envoy
    labels:
      app: envoy
  data:
    envoy.yaml: |
      static_resources:
        clusters:
        - name: news-api
          connect_timeout: 0.25s
          type: strict_dns
          lb_policy: round_robin
          load_assignment:
            cluster_name: news-api
            endpoints:
            - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 127.0.0.1
                      port_value: 3000
        - name: authorino
          connect_timeout: 0.25s
          type: strict_dns
          lb_policy: round_robin
          http2_protocol_options: {}
          load_assignment:
            cluster_name: authorino
            endpoints:
            - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: authorino-authorino-authorization.news-agency.svc.cluster.local
                      port_value: 50051
        listeners:
        - address:
            socket_address:
              address: 0.0.0.0
              port_value: 8080
          filter_chains:
          - filters:
            - name: envoy.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: local
                route_config:
                  name: local_route
                  virtual_hosts:
                  - name: local_service
                    domains: ['*']
                    routes:
                    - match: { prefix: / }
                      route: { cluster: news-api }
                http_filters:
                - name: envoy.filters.http.ext_authz
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                    transport_api_version: V3
                    failure_mode_allow: false
                    include_peer_certificate: true
                    grpc_service:
                      envoy_grpc: { cluster_name: authorino }
                      timeout: 1s
                - name: envoy.filters.http.router
                  typed_config: {}
                use_remote_address: true
                skip_xff_append: true
      admin:
        access_log_path: "/tmp/admin_access.log"
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 8081
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20envoy-configmap.yaml)</sub>

  ```sh
  kubectl patch -n news-agency deployment/news-api \
    --type=strategic \
    --patch '{"spec":{"template":{"spec":{"containers":[{"name":"envoy","image":"envoyproxy/envoy:v1.19-latest","command":["/usr/local/bin/envoy"],"args":["--config-path /usr/local/etc/envoy/envoy.yaml","--service-cluster front-proxy","--log-level info","--component-log-level filter:trace,http:debug,router:debug"],"ports":[{"containerPort":8080}],"volumeMounts":[{"mountPath":"/usr/local/etc/envoy","name":"config","readOnly":true}]}],"volumes":[{"configMap":{"items":[{"key":"envoy.yaml","path":"envoy.yaml"}],"name":"envoy"},"name":"config"}]}}}}'
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20patch%20-n%20news-agency%20deployment/news-api%20--type=strategic%20--patch%20%22$(cat%20envoy-sidecar-patch.yaml)%22)</sub>
</details>

<details>
  <summary><b>③ Re-write the service</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: news-api
    labels:
      app: news-api
  spec:
    selector:
      app: news-api
    ports:
    - name: http
      port: 8080
      protocol: TCP
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20news-api-service.yaml)</sub>

  ```sh
  kill $PID; kubectl port-forward -n news-agency service/news-api 8080:8080 2>&1 >/dev/null &;PID=$!
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kill%20$PID;%20kubectl%20port-forward%20-n%20news-agency%20service/news-api%208080:8080%202%3E%261%20%3E/dev/null%20%26;PID=$!)</sub>
</details>

<details>
  <summary><b>④ Try the application locked down</b></summary>

  ```sh
  curl http://api.news-agency.127.0.0.1.nip.io:8080/tech -s -i
  # HTTP/1.1 404 Not Found
  # x-ext-auth-reason: Service not found
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20http://api.news-agency.127.0.0.1.nip.io:8080/tech%20-s%20-i)</sub>
</details>

<details>
  <summary><b>⑤ Create an <code>AuthConfig</code> custom resource</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: authorino.kuadrant.io/v1beta1
  kind: AuthConfig
  metadata:
    name: news-api
  spec:
    hosts:
    - api.news-agency.127.0.0.1.nip.io
    identity:
    - name: api-key-users
      apiKey:
        selector:
          matchLabels:
            app: news-api
      credentials:
        in: authorization_header
        keySelector: API-KEY
    authorization:
    - name: k8s-rbac
      kubernetes:
        user:
          valueFrom:
            authJSON: auth.identity.metadata.annotations.api\.news-agency/username
        resourceAttributes:
          namespace:
            value: news-agency
          group:
            value: api.news-agency/v1
          resource:
            value: news
          verb:
            valueFrom:
              authJSON: context.request.http.method.@case:lower
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20authconfig-1.yaml)</sub>
</details>

## Try the application protected

<details>
  <summary><b>① Try the application missing authentication</b></summary>

  ```sh
  curl http://api.news-agency.127.0.0.1.nip.io:8080/tech -s -i
  # HTTP/1.1 401 Unauthorized
  # www-authenticate: API-KEY realm="api-key-users"
  # x-ext-auth-reason: credential not found
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20http://api.news-agency.127.0.0.1.nip.io:8080/tech%20-s%20-i)</sub>
</details>

<details>
  <summary><b>② Create an API key</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: v1
  kind: Secret
  metadata:
    name: api-key-1
    labels:
      authorino.kuadrant.io/managed-by: authorino
      app: news-api
    annotations:
      api.news-agency/username: alice
      api.news-agency/name: Alice Smith
  stringData:
    api_key: ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx
  type: Opaque
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20api-key-secret.yaml)</sub>
</details>

<details>
  <summary><b>③ Try the application missing authorization</b></summary>

  ```sh
  curl -H 'Authorization: API-KEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' http://api.news-agency.127.0.0.1.nip.io:8080/tech -s -i
  # HTTP/1.1 403 Forbidden
  # x-ext-auth-reason: Not authorized
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20API-KEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20http://api.news-agency.127.0.0.1.nip.io:8080/tech%20-s%20-i)</sub>
</details>

<details>
  <summary><b>④ Grant <i>writer</i> access to the user</b></summary>

  Create the roles:

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: news-writer
  rules:
  - apiGroups: ["api.news-agency/v1"]
    resources: ["news"]
    verbs: ["get", "post"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: news-admin
  rules:
  - apiGroups: ["api.news-agency/v1"]
    resources: ["news"]
    verbs: ["get", "post", "delete"]
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-agency%20apply%20-f%20roles.yaml)</sub>

  Bind the user to the role:

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: news-writers
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: news-writer
  subjects:
  - kind: User
    name: alice
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-agency%20apply%20-f%20news-writers-rolebinding.yaml)</sub>
</details>

<details>
  <summary><b>⑤ Try the application with <i>writer</i> access</b></summary>

  Try to read the news articles:

  ```sh
  curl -H 'Authorization: API-KEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' http://api.news-agency.127.0.0.1.nip.io:8080/tech -s -i
  # HTTP/1.1 200 OK
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20API-KEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20http://api.news-agency.127.0.0.1.nip.io:8080/tech%20-s%20-i)</sub>

  Try to create a news article:

  ```sh
  curl -H 'Authorization: API-KEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' \
       -X POST \
       -d '{"title":"Pegasus spyware was used to hack reporters’ phones. I’m suing its creators","body":"When you’re infected by Pegasus, spies effectively hold a clone of your phone – we’re fighting back (By Nelson Rauda Zablah - The Guardian)"}' \
       http://api.news-agency.127.0.0.1.nip.io:8080/tech -s -i
  # HTTP/1.1 200 OK
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20API-KEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20-X%20POST%20-d%20'%7B%22title%22:%22Pegasus%20spyware%20was%20used%20to%20hack%20reporters%E2%80%99%20phones.%20I%E2%80%99m%20suing%20its%20creators%22,%22body%22:%22When%20you%E2%80%99re%20infected%20by%20Pegasus,%20spies%20effectively%20hold%20a%20clone%20of%20your%20phone%20%E2%80%93%20we%E2%80%99re%20fighting%20back%20(By%20Nelson%20Rauda%20Zablah%20-%20The%20Guardian)%22%7D'%20http://api.news-agency.127.0.0.1.nip.io:8080/tech%20-s%20-i)</sub>

  Try to delete a news article:

  ```sh
  article_id=$(curl -H 'Authorization: API-KEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' http://api.news-agency.127.0.0.1.nip.io:8080/tech -s | jq -r '.[1].id')
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$article_id=$(curl%20-H%20'Authorization:%20API-KEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20http://api.news-agency.127.0.0.1.nip.io:8080/tech%20-s%20%7C%20jq%20-r%20'.%5B1%5D.id');echo%20$article_id)</sub>

  ```sh
  curl -H 'Authorization: API-KEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' \
       -X DELETE \
       http://api.news-agency.127.0.0.1.nip.io:8080/tech/$article_id -s -i
  # HTTP/1.1 403 Forbidden
  # x-ext-auth-reason: Not authorized
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20API-KEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20-X%20DELETE%20http://api.news-agency.127.0.0.1.nip.io:8080/tech/$article_id%20-s%20-i)</sub>
</details>

## Extend to service accounts

Extend access to the application to Kubernetes service accounts, authenticated using Kubernetes TokenReview.

<details>
  <summary><b>① Alter the <code>AuthConfig</code> custom resource</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: authorino.kuadrant.io/v1beta1
  kind: AuthConfig
  metadata:
    name: news-api
  spec:
    hosts:
    - api.news-agency.127.0.0.1.nip.io
    identity:
    - name: api-key-users
      apiKey:
        selector:
          matchLabels:
            app: news-api
      credentials:
        in: authorization_header
        keySelector: API-KEY
      extendedProperties:
      - name: username
        valueFrom:
          authJSON: auth.identity.metadata.annotations.api\.news-agency/username
    - name: k8s-tokens
      kubernetes:
        audiences:
        - https://kubernetes.default.svc.cluster.local
      extendedProperties:
      - name: username
        valueFrom:
          authJSON: auth.identity.sub
    authorization:
    - name: k8s-rbac
      kubernetes:
        user:
          valueFrom:
            authJSON: auth.identity.username
        resourceAttributes:
          namespace:
            value: news-agency
          group:
            value: api.news-agency/v1
          resource:
            value: news
          verb:
            valueFrom:
              authJSON: context.request.http.method.@case:lower
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20authconfig-2.yaml)</sub>
</details>

<details>
  <summary><b>② Create a service account</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: news-api-user-janitor
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20service-account.yaml)</sub>
</details>

<details>
  <summary><b>③ Grant <i>admin</i> access to the service account</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: news-admins
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: news-admin
  subjects:
  - kind: ServiceAccount
    name: news-api-user-janitor
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20-n%20news-agency%20apply%20-f%20news-admins-rolebinding.yaml)</sub>
</details>

<details>
  <summary><b>④ Request a token</b></summary>

  ```sh
  access_token=$(kubectl create -n news-agency token news-api-user-janitor)
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$access_token=$(kubectl%20create%20-n%20news-agency%20token%20news-api-user-janitor);echo%20$access_token)</sub>
</details>

<details>
  <summary><b>⑤ Try the application with <i>admin</i> access</b></summary>

  Try to read the news articles:

  ```sh
  curl -H "Authorization: Bearer $access_token" http://api.news-agency.127.0.0.1.nip.io:8080/tech -s -i
  # HTTP/1.1 200 OK
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20%22Authorization:%20Bearer%20$access_token%22%20http://api.news-agency.127.0.0.1.nip.io:8080/tech%20-s%20-i)</sub>

  Try to delete a news article:

  ```sh
  curl -H "Authorization: Bearer $access_token" \
       -X DELETE \
       http://api.news-agency.127.0.0.1.nip.io:8080/tech/$article_id -s -i
  # HTTP/1.1 200 OK
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20%22Authorization:%20Bearer%20$access_token%22%20-X%20DELETE%20http://api.news-agency.127.0.0.1.nip.io:8080/tech/$article_id%20-s%20-i)</sub>
</details>

## Bonus: Inject `author` and `user_id`

<details>
  <summary><b>① Alter the <code>AuthConfig</code> custom resource</b></summary>

  ```sh
  kubectl apply -n news-agency -f -<<EOF
  apiVersion: authorino.kuadrant.io/v1beta1
  kind: AuthConfig
  metadata:
    name: news-api
  spec:
    hosts:
    - api.news-agency.127.0.0.1.nip.io
    identity:
    - name: api-key-users
      apiKey:
        selector:
          matchLabels:
            app: news-api
      credentials:
        in: authorization_header
        keySelector: API-KEY
      extendedProperties:
      - name: username
        valueFrom:
          authJSON: auth.identity.metadata.annotations.api\.news-agency/username
      - name: name
        valueFrom:
          authJSON: auth.identity.metadata.annotations.api\.news-agency/name
    - name: k8s-tokens
      kubernetes:
        audiences:
        - https://kubernetes.default.svc.cluster.local
      extendedProperties:
      - name: username
        valueFrom:
          authJSON: auth.identity.sub
      - name: name
        valueFrom:
          authJSON: auth.identity.serviceaccount.name
    authorization:
    - name: k8s-rbac
      kubernetes:
        user:
          valueFrom:
            authJSON: auth.identity.username
        resourceAttributes:
          namespace:
            value: news-agency
          group:
            value: api.news-agency/v1
          resource:
            value: news
          verb:
            valueFrom:
              authJSON: context.request.http.method.@case:lower
    response:
    - name: x-ext-auth-data
      json:
        properties:
        - name: author
          valueFrom:
            authJSON: auth.identity.name
        - name: user_id
          valueFrom:
            authJSON: auth.identity.username
  EOF
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kubectl%20apply%20-n%20news-agency%20-f%20authconfig-3.yaml)</sub>
</details>

<details>
  <summary><b>② Create a news article</b></summary>

  ```sh
  curl -H 'Authorization: API-KEY ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx' \
       -X POST \
       -d '{"title":"Airlines warn of higher fares as industry moves to net zero target","body":"Iata director general Willie Walsh calls for greater production of sustainable aviation fuel (By Reuters)"}' \
       http://api.news-agency.127.0.0.1.nip.io:8080/business -s -i
  # HTTP/1.1 200 OK
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$curl%20-H%20'Authorization:%20API-KEY%20ndyBzreUzF4zqDQsqSPMHkRhriEOtcRx'%20-X%20POST%20-d%20'%7B%22title%22:%22Airlines%20warn%20of%20higher%20fares%20as%20industry%20moves%20to%20net%20zero%20target%22,%22body%22:%22Iata%20director%20general%20Willie%20Walsh%20calls%20for%20greater%20production%20of%20sustainable%20aviation%20fuel%20(By%20Reuters)%22%7D'%20http://api.news-agency.127.0.0.1.nip.io:8080/business%20-s%20-i)</sub>
</details>

## Cleanup

<details>
  <summary><b>① Delete the cluster</b></summary>

  ```sh
  kind delete cluster --name demo
  ```

  <sub>[▶︎ Run in terminal](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=demo$$kind%20delete%20cluster%20--name%20demo)</sub>
</details>
