# mongodb and mongo express k8 deployment

Setup a mongoDB app and mongo express web-app using a k8 statefulset

1. Create mongoDB pod.

Check DockerHub for how to use the image (ports, pwds, etc). Leave username & pwd value empty b/c configMaps are checked into code repos.

example Pod config:

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
