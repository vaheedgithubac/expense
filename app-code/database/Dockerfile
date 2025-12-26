# docker build app-code/database -t database-img:1.0.0 -f Docker/Dockerfile
FROM mysql:8.0
# ENV MYSQL_ROOT_PASSWORD=ExpenseApp@1
COPY Docker_Stuff/mysql_scripts/*.sql /docker-entrypoint-initdb.d
