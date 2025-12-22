## ✅ Option 1: Smallest + safest
**`cgr.dev/chainguard/mysql-client:8.0` (true MySQL 8 client)**  
🔹 When to use  
- You want exact MySQL 8 client
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
