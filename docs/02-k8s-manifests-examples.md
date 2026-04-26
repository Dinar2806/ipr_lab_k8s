# Примеры Kubernetes-манифестов

Этот документ содержит учебные шаблоны с намеренно незаполненными полями. Их нельзя просто применить в кластер без ручной доработки.

## Важно: обязательные ручные правки

Перед запуском вы обязаны заменить все значения вида:

- `<...>`
- `TODO_*`
- `CHANGEME`

Минимальный набор правок:

1. namespace, hostnames и имена ресурсов под вашу работу;
2. секреты и DSN (без дефолтных значений из примеров);
3. теги образов (не `latest` для `prod`);
4. storage class/размера томов под ваш кластер;
5. probes/resources/affinity в соответствии с заданием.

## 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <team-namespace>
```

## 2. ConfigMap и Secret

### Общий ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: messager-config
  namespace: <team-namespace>
data:
  FRONTEND_BFF_URL: "TODO_PUBLIC_BFF_URL_OR_EMPTY"
  FRONTEND_BFF_INTERNAL_URL: "http://bff:8080"
  BFF_HTTP_PORT: "8080"
  USER_HTTP_PORT: "8081"
  MSG_HTTP_PORT: "8082"
  USER_SERVICE_URL: "http://user-service:8081"
  MSG_SERVICE_URL: "http://message-service:8082"
  MSG_UPLOADS_DIR: "/app/uploads"
  POSTGRES_DB: "TODO_POSTGRES_BOOTSTRAP_DB"
  POSTGRES_HOST: "postgres"
  POSTGRES_PORT: "5432"
```

### Secret для БД

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: messager-db-secret
  namespace: <team-namespace>
type: Opaque
stringData:
  POSTGRES_USER: "CHANGEME_DB_USER"
  POSTGRES_PASSWORD: "CHANGEME_DB_PASSWORD"
  USER_DB_DSN: "host=postgres user=CHANGEME_DB_USER password=CHANGEME_DB_PASSWORD dbname=<users_db_name> sslmode=disable"
  MSG_DB_DSN: "host=postgres user=CHANGEME_DB_USER password=CHANGEME_DB_PASSWORD dbname=<messages_db_name> sslmode=disable"
```

## 3. Postgres

### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: <team-namespace>
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: <your-storage-class>
  resources:
    requests:
      storage: <size-like-5Gi>
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: <team-namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: POSTGRES_DB
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U $POSTGRES_USER"]
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            requests:
              cpu: "TODO_CPU_REQUEST"
              memory: "TODO_MEMORY_REQUEST"
            limits:
              cpu: "TODO_CPU_LIMIT"
              memory: "TODO_MEMORY_LIMIT"
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-pvc
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: <team-namespace>
spec:
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

## 4. Миграции (Jobs)

### migrate-users

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-users
  namespace: <team-namespace>
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate-users
          image: ghcr.io/kukymbr/goose-docker:latest
          env:
            - name: GOOSE_DRIVER
              value: postgres
            - name: GOOSE_DBSTRING
              value: "host=postgres user=<db_user> password=<db_password> dbname=<users_db_name> sslmode=disable"
            - name: GOOSE_MIGRATION_DIR
              value: /migrations
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: users-migrations-cm
```

### migrate-messages

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-messages
  namespace: <team-namespace>
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate-messages
          image: ghcr.io/kukymbr/goose-docker:latest
          env:
            - name: GOOSE_DRIVER
              value: postgres
            - name: GOOSE_DBSTRING
              value: "host=postgres user=<db_user> password=<db_password> dbname=<messages_db_name> sslmode=disable"
            - name: GOOSE_MIGRATION_DIR
              value: /migrations
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: messages-migrations-cm
```

> В реальном решении миграции удобнее запускать как `initContainer` или отдельным CI/CD этапом. Для лабораторной подходит Job.

## 5. user-service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: <team-namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: mablinov2704/user-service:<tag-for-env>
          ports:
            - containerPort: 8081
          env:
            - name: HTTP_PORT
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: USER_HTTP_PORT
            - name: DB_DSN
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: USER_DB_DSN
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: <team-namespace>
spec:
  selector:
    app: user-service
  ports:
    - name: http
      port: 8081
      targetPort: 8081
```

## 6. message-service (с volume для файлов)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: message-service
  namespace: <team-namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: message-service
  template:
    metadata:
      labels:
        app: message-service
    spec:
      containers:
        - name: message-service
          image: mablinov2704/message-service:<tag-for-env>
          ports:
            - containerPort: 8082
          env:
            - name: HTTP_PORT
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: MSG_HTTP_PORT
            - name: DB_DSN
              valueFrom:
                secretKeyRef:
                  name: messager-db-secret
                  key: MSG_DB_DSN
            - name: UPLOADS_DIR
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: MSG_UPLOADS_DIR
          volumeMounts:
            - name: uploads
              mountPath: /app/uploads
      volumes:
        - name: uploads
          persistentVolumeClaim:
            claimName: message-uploads-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: message-service
  namespace: <team-namespace>
spec:
  selector:
    app: message-service
  ports:
    - name: http
      port: 8082
      targetPort: 8082
```

## 7. bff

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bff
  namespace: <team-namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bff
  template:
    metadata:
      labels:
        app: bff
    spec:
      containers:
        - name: bff
          image: mablinov2704/bff:<tag-for-env>
          ports:
            - containerPort: 8080
          env:
            - name: HTTP_PORT
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: BFF_HTTP_PORT
            - name: USER_SERVICE_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: USER_SERVICE_URL
            - name: MSG_SERVICE_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: MSG_SERVICE_URL
---
apiVersion: v1
kind: Service
metadata:
  name: bff
  namespace: <team-namespace>
spec:
  selector:
    app: bff
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

## 8. frontend + Ingress (Ingress опционален)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: <team-namespace>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: mablinov2704/frontend:<tag-for-env>
          ports:
            - containerPort: 80
          env:
            - name: BFF_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: FRONTEND_BFF_URL
            - name: BFF_INTERNAL_URL
              valueFrom:
                configMapKeyRef:
                  name: messager-config
                  key: FRONTEND_BFF_INTERNAL_URL
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: <team-namespace>
spec:
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <team-ingress-name>
  namespace: <team-namespace>
spec:
  ingressClassName: <your-ingress-class>
  rules:
    - host: <your-hostname>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

> Ingress можно не использовать: для лабораторной допустим внешний доступ через `NodePort` или `LoadBalancer`.

## 9. Пример kustomization (base)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: <team-namespace>
resources:
  - namespace.yaml
  - configmap.yaml
  - secret.yaml
  - postgres-pvc.yaml
  - postgres.yaml
  - migrate-users-job.yaml
  - migrate-messages-job.yaml
  - user-service.yaml
  - message-service.yaml
  - bff.yaml
  - frontend.yaml
  # ingress.yaml (опционально)
```
