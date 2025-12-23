## ✅ Option 1: Smallest + safest
**`cgr.dev/chainguard/mysql-client:8.0` (true MySQL 8 client)**  

**🔹 When to use**  
- You want **exact MySQL 8 client**
- Very small image
- Security-focused production setup
## ⚠️ Important
Chainguard images are **distroless → NO shell (`sh`)**.  
So we must **run `mysql` directly**, not a shell loop.

## Kubernetes initContainer (Chainguard)
```yaml
initContainers:
  - name: wait-for-mysql
    image: cgr.dev/chainguard/mysql-client:8.0
    env:
      - name: MYSQL_HOST
        value: mysql            # Your MySQL service name
      - name: MYSQL_PORT
        value: "3306"
      - name: MYSQL_USER
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: username
      - name: MYSQL_PWD            # Secure way to pass password
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: password
    command: ["mysql","-h$(MYSQL_HOST)","-P$(MYSQL_PORT)","-u$(MYSQL_USER)","-e","SELECT 1"]
```
**Very Important Note:** Variable name should be `MYSQL_PWD`, since mysql client reads that env variable by default
Note:
- use `$(MYSQL_HOST)` but not use `${MYSQL_HOST}`, since `cgr.dev/chainguard/mysql-client:8.0` has no shell

## ✅ Key Points  
**1. Password handling is safe**  
- Uses the `MYSQL_PWD` environment variable.  
- Password does **not appear in command-line arguments** or the process list.  

**2. No shell required**  
- Chainguard images are distroless.  
- No `sh` or loops — Kubernetes will **retry the initContainer** automatically if it fails.  

**3. Minimal and production-ready**  
- Small image (~10 MB).  
- Only contains the MySQL client.

## 🧠 Behavior  
- Kubernetes retries the init container until MySQL accepts connections  
- No loops needed  
- Clean and idiomatic Kubernetes pattern  
✔ Small  
✔ Secure  
✔ Production-grade  

## ✅ Option 2: No external registry  
**Alpine + MariaDB mysql-client**  
**🔹 When to use**  
- You want to use only Docker Hub.  
- You want a shell for loops and logging.  
- MySQL 8 compatibility is sufficient (MariaDB client).  

## Dockerfile (pin Alpine version!)   
```text
FROM alpine:3.19
RUN apk add --no-cache mysql-client
ENTRYPOINT ["sh", "-c"]
```
**🔐 Why pin Alpine?
`alpine:latest` can change MariaDB versions unexpectedly.

**Kubernetes initContainer (Alpine)
```yaml
initContainers:
  - name: wait-for-mysql
    image: alpine:3.19  # lightweight image with shell
    env:
      - name: MYSQL_HOST
        value: mysql          # your MySQL service name
      - name: MYSQL_PORT
        value: "3306"
      - name: MYSQL_USER
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: username
      - name: MYSQL_PWD          # secure way to pass password
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: password
    command: ["sh","-c","for i in $(seq 1 30); do mysql -h $MYSQL_HOST -P $MYSQL_PORT -u$MYSQL_USER -e 'SELECT 1' && exit 0; echo Waiting for MySQL...; sleep 2; done; echo Timeout waiting for MySQL; exit 1"]
```
**Very Important Note:** Variable name should be `MYSQL_PWD`, since mysql client reads that env variable by default  

## ✅ How it works  
**1.** Tries **30 times** `(seq 1 30)`, sleeping **2 seconds** between attempts → ~60 s total timeout.  
**2**. Uses `MYSQL_PWD` from the environment, **no password exposed in process list**.  
**3**. Logs `"Waiting for MySQL..."` on each failed attempt.  
**4**. Exits **1** if MySQL never becomes available → pod fails init container (good for readiness gating).  

## 🧠 Behavior
- Loops until DB is reachable  
- Clear logs  
- Very easy to debug  
✔ Simple  
✔ No external registry   
✔ Slightly larger image (~15–20MB)  

## Full Deployment example
This is production-ready and uses secure password handling (`MYSQL_PWD`)  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      initContainers:
        - name: wait-for-mysql
          image: cgr.dev/chainguard/mysql-client:8.0
          env:
            - name: MYSQL_HOST
              value: mysql
            - name: MYSQL_PORT
              value: "3306"
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: username
            - name: MYSQL_PWD             # secure way to pass password
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          command: ["mysql","-h$(MYSQL_HOST)","-P$(MYSQL_PORT)","-u$(MYSQL_USER)","-e","SELECT 1"]
      containers:
        - name: my-app-container
          image: my-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: MYSQL_HOST
              value: mysql
            - name: MYSQL_PORT
              value: "3306"
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: username
            - name: MYSQL_PWD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          readinessProbe:
            exec:
              command:
                - mysql
                - -h$(MYSQL_HOST)
                - -P$(MYSQL_PORT)
                - -u$(MYSQL_USER)
                - -e
                - "SELECT 1"
            initialDelaySeconds: 5
            periodSeconds: 5
```
## ✅ How this works  
**1. InitContainer** runs first  
- Tries to connect to MySQL using the `SELECT 1` command.  
- If MySQL isn’t ready, it **fails and Kubernetes retries automatically**.  

**2. Main app container** starts **only after the initContainer succeeds**.  

**3. ReadinessProbe** ensures the pod isn’t marked ready until MySQL is actually reachable.  
- Uses the same secure `MYSQL_PWD` approach.  
- Runs every 5 seconds after a 5-second delay.  
## ⚡ Advantages  
  - Minimal, secure, production-ready.   
  - No passwords in command-line args (`MYSQL_PWD` used).  
  - Works with **distroless Chainguard MySQL client**.  
  - Automatic retries handled by Kubernetes; no shell loops needed.  

## 🥉 alpine + mysql-client (ULTRA lightweight)  
```yaml
image: alpine:3.19
command:
  - sh
  - -c
  - |
    apk add --no-cache mysql-client
    until mysqladmin ping -h mysql -u healthcheck -p"$MYSQL_HEALTHCHECK_PASSWORD"; do
      sleep 3
    done
```
