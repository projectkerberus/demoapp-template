# Demo app note

This repo contain the demo app.

## Application source code

For demo app starting point refer to [this](https://github.com/spring-guides/gs-accessing-data-mysql) and with changes explained [here](https://github.com/bitnami/tutorials/blob/master/spring-boot-app/README.md).

Further or Mysql compatibility the line is added in application.properties:

```java
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
```

Generate the WAR file by running the mvn package command. The resulting file will be located in target/gs-mysql-data-0.1.0.war.

## Docker Build

In the app-docker folder you find the Dockerfile containing:

```dockerfile
FROM bitnami/tomcat:9.0
COPY gs-mysql-data-0.1.0.war /opt/bitnami/tomcat/webapps_default/
```

Build:

```bash
docker build -t DOCKER_USERNAME/spring-java-app .
```

Push:

```bash
docker push DOCKER_USERNAME/spring-java-app:latest
```

Docker compose yaml for testing image:

```yaml
version: '2'

services:
    myapp:
    image: 'DOCKER_USERNAME/spring-java-app'
    environment:
      - 'SPRING_APPLICATION_JSON={"spring": {"datasource":{"url": "jdbc:mysql://mariadb:3306/db_example", "username": "springuser", "password": "ThePassword"}}}'
      - ALLOW_EMPTY_PASSWORD=yes
    depends_on:
      - mariadb
    ports:
     - '8080:8080
```

test:

```bash
docker compose-up
curl 'localhost:8080/gs-mysql-data-0.1.0/demo/all'
curl 'localhost:8080/gs-mysql-data-0.1.0/demo/add?name=First&email=someemail@someemailprovider.com'
curl 'localhost:8080/gs-mysql-data-0.1.0/demo/all'
```

## Helm chart

The hem chart in this repo is the result of the 
The add dependency in Chart.yaml (not in requirements fiels since in deprecated, changed to a more recent mariadb image (statesful api version error)):

```yaml
- name: mariadb
  version: 9.4.2
  repository: https://charts.bitnami.com/bitnami
  condition: mariadb.enabled
  tags:
    - spring-java-app-database
```

since this mariadb cannot receive values in dependency, untar mariadb under chart, add the required parameter to create the db, the user and passw:

```yaml
auth:
  database: db_example
  username: "springuser"
  password: "ThePassword"
```

modify in values:

```yaml
image:
   registry: docker.io
   repository: DOCKER_USERNAME/spring-java-app
   tag: latest
```

add to values.yaml:

```yaml
mariadb:

  enabled: true
  replication:
    enabled: false
    
  db:
    name: db_example
    user: springuser
    password: ThePassword
```

add to _helpers.tpl

```yaml
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "mariadb.fullname" -}}
{{- printf "%s-%s" .Release.Name "mariadb" | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

create spring-secret.yaml in template directory:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ template "tomcat.fullname" . }}-spring
  labels:
    app: {{ template "tomcat.fullname" . }}
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
type: Opaque
data:
  spring-db: {{ printf "{\"spring\": {\"datasource\":{\"url\": \"jdbc:mysql://%s:3306/%s\", \"username\": \"%s\", \"password\": \"%s\"}}}" (include "mariadb.fullname" .) .Values.mariadb.db.name .Values.mariadb.db.user .Values.mariadb.db.password | b64enc }}
```

add to _pod.tpl

```yaml
 - name: SPRING_APPLICATION_JSON
    valueFrom:
      secretKeyRef:
        name: {{ template "common.names.fullname" . }}-spring
        key: spring-db
```

change in values.yaml form LoadBalancer to NodePort:

```yaml
service:
  ## @param service.type Kubernetes Service type
  ## For minikube, set this to NodePort, elsewhere use LoadBalancer
  ##
  type: NodePort
  ## @param service.port Service HTTP port
  ##
  port: 80
  ## @param service.nodePort Kubernetes http node port
  ## ref: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
  ##
  nodePort: "30080"
```

and for Kuma ingress:

```yaml
ingress:
  ## @param ingress.enabled Enable ingress controller resource
  ##
  enabled: true
  ## @param ingress.certManager Set this to true in order to add the corresponding annotations for cert-manager
  ##
  certManager: true
  ## @param ingress.hostname Default host for the ingress resource
  ##
  hostname: app.projectx.kiratech.biz
  ## @param ingress.annotations Ingress annotations
  ## For a full list of possible ingress annotations, please see
  ## ref: https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md
  ##
  ## If certManager is set to true, annotation kubernetes.io/tls-acme: "true" will automatically be set
  ##
  annotations:
    kubernetes.io/ingress.class: "kong"
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
  ## @param ingress.tls Enable TLS configuration for the hostname defined at `ingress.hostname` parameter
  ## TLS certificates will be retrieved from a TLS secret with name: {{- printf "%s-tls" .Values.ingress.hostname }}
  ## You can use the ingress.secrets parameter to create this TLS secret, relay on cert-manager to create it, or
  ## let the chart create self-signed certificates for you
  ##
  tls: true
```

run:

```bash
helm dependency update .
helm install -n app --create-namespace  spring-java .
```

test:

```html
https://app.projectx.kiratech.biz/gs-mysql-data-0.1.0/demo/add?name=First&email=someemail@someemailprovider.com
https://app.projectx.kiratech.biz/gs-mysql-data-0.1.0/demo/all
```
