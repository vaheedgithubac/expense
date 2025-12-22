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
        value: mysql
      - name: MYSQL_PORT
        value: "3306"
      - name: MYSQL_USER
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: username
      - name: MYSQL_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: password
    command:
      - mysql
      - -h$(MYSQL_HOST)
      - -P$(MYSQL_PORT)
      - -u$(MYSQL_USER)
      - -p$(MYSQL_PASSWORD)
      - -e
      - "SELECT 1"  
```
## 🧠 Behavior  
- Kubernetes retries the init container until MySQL accepts connections  
- No loops needed  
- Clean and idiomatic Kubernetes pattern  
✔ Small  
✔ Secure  
✔ Production-grade  

✅ Option 2: No external registry  
**Alpine + MariaDB mysql-client**  
**🔹 When to use  
- You want only Docker Hub  
- You want a shell for loops/logging  
- MySQL 8 compatibility is sufficient (MariaDB client)  

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
    image: my-alpine-mysql-client:1.0
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
      - name: MYSQL_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: password
    command:
      - sh
      - -c
      - |
        until mysql -h "$MYSQL_HOST" -P "$MYSQL_PORT" \
          -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" \
          -e "SELECT 1"; do
          echo "Waiting for MySQL..."
          sleep 2
        done
```
##🧠 Behavior
- Loops until DB is reachable  
- Clear logs  
- Very easy to debug  
✔ Simple  
✔ No external registry   
✔ Slightly larger image (~15–20MB)  
