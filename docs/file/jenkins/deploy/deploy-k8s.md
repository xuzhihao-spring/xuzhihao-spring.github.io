```yml
---
apiVersion: v1
kind: Service
metadata:
  name: eureka
  labels:
    app: eureka
spec:
  type: NodePort
  ports:
    - port: 10086
      name: eureka
      targetPort: 10086
  selector:
    app: eureka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka
spec:
  serviceName: "eureka"
  replicas: 2
  selector:
    matchLabels:
      app: eureka
  template:
    metadata:
      labels:
        app: eureka
    spec:
      imagePullSecrets:
        - name: $SECRET_NAME
      containers:
        - name: eureka
          image: $IMAGE_NAME
          ports:
            - containerPort: 10086
          env:
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: EUREKA_SERVER
              value: "http://eureka-0.eureka:10086/eureka/,http://eureka-1.eureka:10086/eureka/"
            - name: EUREKA_INSTANCE_HOSTNAME
              value: ${MY_POD_NAME}.eureka
  podManagementPolicy: "Parallel"
```