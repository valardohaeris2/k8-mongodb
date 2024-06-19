# MongoDB and Mongo Express k8 Deployment

## Pre-reqs:

- Minikube installed and started
- Docker Desktop running
- Helm CLI
- (optional) add an alias to your zsh/bash profile where `alias k=kubectl`. This will prevent you from having to type `kubectl` and instead use `k`.

## Overview:

Setup a mongoDB app and mongo express web-app using a k8 statefulset.

**The client request flow is as follows:**  
request comes from browser >> through External Service of mongo-express >> fwds traffic to mongo-express pod >> mongo-express pod connects to Internal Service of mongoDB (dB URL) >> mongoDB Internal Service fwds traffic to the mongoDB pod >> authentication using the Secret

## Workflow

## 1. Create mongoDB pod

Check DockerHub for how to use the image (ports, pwds, etc). Leave username & pwd value empty. A configMap & Secret will be created to reference these values.

**example mongoDB Pod config:**

```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mongodb-deployment
      labels:
        app: mongodb
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mongodb
      template:
        metadata:
          labels:
            app: mongodb
        spec:
          containers:
          - name: mongodb
            image: mongo
            resources:
              limits:
                memory: "128Mi"
                cpu: "500m"
            ports:
            - containerPort: 27017
            env:
            - name: MONGO_ROOT_USERNAME
              value:
            - name: MONGO_ROOT_PASSWORD
          value:
```

## 2. Create Secrets that contain dB username and password authN credentials

**a. Create a `mongo-secret.yaml`**

```
 apiVersion: v1
 kind: Secret
 metadata:
   name: mongodb-secret
 type: Opaque
 data:
   mongo-root-username:
   mongo-root-password:
```

**b. base64 encode the username:**

`echo -n 'arya stark' | base64`

**c. base64 encode password:**

`echo -n 'needle' | base64`

**d. paste the encoded values into the `mongo-secret.yaml`:**

```
    apiVersion: v1
    kind: Secret
    metadata:
      name: mongodb-secret
    type: Opaque
    data:
      mongo-root-username: YX....          <=paste
      mongo-root-password: bmVl....        <=paste
```

**e. Create the secret from the secret configFile:**

       The secret must be created before deploying the mongoDB Pod.

`kubectl -f mongo-secret.yaml` or `k -f mongo-secret.yaml`

**f. Reference the secret values in the `mongo.yaml` configfile:**

```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mongodb-deployment
      labels:
        app: mongodb
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mongodb
      template:
        metadata:
          labels:
            app: mongodb
        spec:
          containers:
            - name: mongodb
              image: mongo
              resources:
                limits:
                  memory: "128Mi"
                  cpu: "500m"
              ports:
                - containerPort: 27017
              env:
                - name: MONGO_ROOT_USERNAME
                  valueFrom:
                    secretKeyRef:                  <= Reference username
                      name: mongodb-secret
                      key: mongo-root-username
                - name: MONGO_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:                  <= Reference password
                      name: mongodb-secret
                  key: mongo-root-password
```

**g. Deploy the `mongo.yaml`:**

`k apply -f mongo.yaml`
`k get pod`

## 3. Create a mongodb Internal Service to allow communication w/ the mongoDB Pod

Use "selector" of the Service and the "label" of the Pod to connect the Pod to the Service.

```
    ---                  <= Add the separator
    apiVersion: v1
    kind: Service
    metadata:
      name: mongodb-service
    spec:
      selector:
        app: mongodb
      ports:
        - port: 27017
          targetPort: 27017
```

`k apply -f mongo.yaml`

## 4. Create mongo-express deployment:

Pass the mongoDB username/password credentials & dB URL to Mongo Express by using the mongo-express deployment.yaml file, env-vars and a ConfigMap

**a. Create Deployment configFile `mongo-express.yaml`:**

        ```
              apiVersion: apps/v1
              kind: Deployment
              metadata:
                name: mongo-express
              spec:
                replicas: 1
                selector:
                  matchLabels:
                    app: mongo-express
                template:
                  metadata:
                    labels:
                      app: mongo-express
                  spec:
                    containers:
                      - name: mongo-express
                        image: mongo-express:1.0.2
                        resources:
                          limits:
                            memory: "128Mi"
                            cpu: "500m"
                        ports:
                          - containerPort: 8081
                        env:
                          - name: ME_CONFIG_MONGODB_ADMIN_USERNAME
                            valueFrom:
                              secretKeyRef:
                                name: mongodb-secret
                                key: mongo-root-username
                          - name: ME_CONFIG_MONGODB_ADMIN_PASSWORD
                            valueFrom:
                              secretKeyRef:
                                name: mongodb-secret
                                key: mongo-root-username
                          - name: ME_CONFIG_MONGODB_SERVER
        ```

**b. Create a `mongo-configmap.yaml` to reference the dB URL:**

    ```
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: mongodb-configmap
          data:
            database_url: mongodb-service
    ```

**c. Reference the configMap w/i the env section of `mongo-express.yaml`:**

    ```
     - name: ME_CONFIG_MONGODB_SERVER
       valueFrom:
         configMapKeyRef:
           name: mongodb-configmap
           key: database_url
    ```

**d. Apply the `mongo-configmap.yaml` & `mongo-express.yaml`:**

`k apply -f mongo-configmap.yaml`
