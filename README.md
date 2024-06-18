# MongoDB and Mongo Express web-app on k8

Setup a mongoDB app and mongo express web-app using a k8 statefulset.

The client request flow is as follows:  
request comes from browser >> through External Service of mongo-express >> fwds traffic to mongo-express pod >> mongo-express pod connects to Internal Service of mongoDB (dB URL) >> mongoDB Internal Service fwds traffic to the mongoDB pod >> authentication using the Secret

1. **Create mongoDB pod**

Check DockerHub for how to use the image (ports, pwds, etc). Leave username & pwd value empty. A configMap & Secret will be created to reference these values.

## example Pod config:

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

2. **Create Secrets that contain dB username and password authN credentials**

## a. Create a mongo-secret.yaml

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

## b. base64 encode the username:

`echo -n 'arya stark' | base64`

## c. base64 encode password:

`echo -n 'needle' | base64`

## d. paste the encoded values into the mongo-secret.yaml:

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

## e. Create the secret from the secret configFile:

    `k apply -f mongo-secret.yaml`
