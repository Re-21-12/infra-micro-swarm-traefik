# infra-micro-swarm-traefik
```
version: '3.9'

services:
  ## 1. SERVICIOS EXPUESTOS (CON TRAEFIK Y HTTPS)

  frontend:
    image: ${FRONTEND_IMAGE}
    # Se eliminan container_name, restart, y ports externos
    depends_on:
      - backend
    networks:
      - server
      - traefik-public
    deploy:
      mode: replicated
      replicas: 2 # ✅ 2 Réplicas para alta disponibilidad
      labels:
        - traefik.enable=true
        # Enrutador: usa el entrypoint websecure (HTTPS)
        - traefik.http.routers.frontend.rule=Host(`app.corazondeseda.lat`)
        - traefik.http.routers.frontend.entrypoints=websecure
        - traefik.http.routers.frontend.tls.certresolver=le
        # Servicio: balancea al puerto interno 80
        - traefik.http.services.frontend.loadbalancer.server.port=80
        - traefik.docker.network=traefik-public

  backend:
    image: ${BACKEND_IMAGE}
    # Se eliminan container_name y ports externos
    environment:
      - ASPNETCORE_URLS=${ASPNETCORE_URLS}
      - DOTNET_RUNNING_IN_CONTAINER=${DOTNET_RUNNING_IN_CONTAINER}
      - DOTNET_USE_POLLING_FILE_WATCHER=${DOTNET_USE_POLLING_FILE_WATCHER}
      - ConnectionStrings__Default=Server=sqlserver,1433;Database=${DB_NAME};User Id=appuser;Password=${APP_USER_PASSWORD};TrustServerCertificate=true;
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
      - JWT_IV=${JWT_IV}
    depends_on:
      - sqlserver
      - keycloak
    networks:
      - server
      - traefik-public
    deploy:
      mode: replicated
      replicas: 2 # ✅ 2 Réplicas para alta disponibilidad
      labels:
        - traefik.enable=true
        - traefik.http.routers.backend.rule=Host(`api.corazondeseda.lat`)
        - traefik.http.routers.backend.entrypoints=websecure
        - traefik.http.routers.backend.tls.certresolver=le
        - traefik.http.services.backend.loadbalancer.server.port=8080
        - traefik.docker.network=traefik-public

  keycloak:
    image: ${KEYCLOAK_IMAGE}
    # Se eliminan container_name, restart y ports externos
    environment:
      - KC_DB=postgres
      - KC_DB_URL_HOST=postgres_keycloak
      - KC_DB_USERNAME=${POSTGRES_USER}
      - KC_DB_PASSWORD=${POSTGRES_PASSWORD}
      - KEYCLOAK_ADMIN=${KEYCLOAK_ADMIN}
      - KEYCLOAK_ADMIN_PASSWORD=${KEYCLOAK_ADMIN_PASSWORD}
      - KC_PROXY=edge
      - KC_HTTP_ENABLED=true
      - KC_HOSTNAME=auth.corazondeseda.lat
      - KC_HOSTNAME_STRICT=false
      - KC_HOSTNAME_STRICT_HTTPS=false
    command:
      - start
    depends_on:
      - postgres_keycloak
    networks:
      - server
      - traefik-public
    deploy:
      mode: replicated
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.http.routers.keycloak.rule=Host(`auth.corazondeseda.lat`)
        - traefik.http.routers.keycloak.entrypoints=websecure
        - traefik.http.routers.keycloak.tls.certresolver=le
        - traefik.http.services.keycloak.loadbalancer.server.port=8080
        - traefik.docker.network=traefik-public

  ## 2. SERVICIOS INTERNOS (MODIFICADOS PARA SWARM)

  sqlserver:
    image: ${SQLSERVER_IMAGE}
    # Se eliminan container_name y restart
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=${SA_PASSWORD}
      - MSSQL_PID=Express
    # Se mantienen ports si son necesarios para acceso directo/herramientas
    ports:
      - "${SQLSERVER_PORT}:1433"
    volumes:
      - sql_data:/var/opt/mssql
    networks:
      - server
    deploy:
      mode: replicated
      replicas: 1
      resources:
        limits:
          memory: 4g

  sql-init:
    image: ${SQLTOOLS_IMAGE}
    depends_on:
      - sqlserver
    volumes:
      - ./script:/scripts
    entrypoint: /bin/bash -c "sleep 15; /opt/mssql-tools/bin/sqlcmd -S sqlserver -U sa -P ${SA_PASSWORD} -i /scripts/init.sql"
    networks:
      - server
    # Se añade deploy para evitar que se ejecute sin fin como un servicio normal
    deploy:
      mode: replicated
      replicas: 0 # Debe ser 0 y correr manualmente, o usar una configuración de Job (más avanzada)

  postgres_keycloak:
    image: ${POSTGRES_IMAGE}
    # Se eliminan container_name y restart
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - keycloak_data:/var/lib/postgresql/data
    networks:
      - server
    ports: # Se mantienen ports si son necesarios
      - "${POSTGRES_PORT}:5432"
    healthcheck:
      test: ["CMD-SHELL", "PGPASSWORD=${POSTGRES_PASSWORD} psql -U ${POSTGRES_USER} -d ${POSTGRES_DB} -c 'SELECT 1;' >/dev/null 2>&1"]
      interval: 10s
      timeout: 5s
      retries: 10
    deploy:
      mode: replicated
      replicas: 1

  admin-service:
    image: ${ADMIN_IMAGE}
    # Se elimina container_name
    ports: # Se mantienen ports si son necesarios
      - "${ADMIN_PORT}:3000"
    environment:
      - DB_HOST=postgres_keycloak
      - DB_PORT=5432
      - DB_NAME=${POSTGRES_DB}
      - DB_USER=${POSTGRES_USER}
      - DB_PASS=${POSTS_PASSWORD}
      - DB_TYPE=postgres
      - KEYCLOAK_URL=http://keycloak:8080
      - KEYCLOAK_REALM=master
      - KEYCLOAK_CLIENT_ID=${ADMIN_CLIENT_ID}
      - KEYCLOAK_CLIENT_SECRET=${ADMIN_CLIENT_SECRET}
      - KEYCLOAK_ADMIN_USERNAME=${KEYCLOAK_ADMIN}
      - KEYCLOAK_ADMIN_PASSWORD=${KEYCLOAK_ADMIN_PASSWORD}
      - KEYCLOAK_ADMIN_CLIENT_ID=admin-cli
      - LOG_LEVEL=info
      - PORT=3000
    depends_on:
      - keycloak
      - postgres_keycloak
    networks:
      - server
    deploy:
      mode: replicated
      replicas: 1

  report-service:
    image: ${REPORT_IMAGE}
    # Se eliminan container_name y restart
    environment:
      - FLASK_ENV=${FLASK_ENV}
      - FLASK_APP=${FLASK_APP}
      - MONGO_URI=${MONGO_URI}
    ports: # Se mantienen ports si son necesarios
      - "${REPORT_PORT}:5000"
    depends_on:
      - backend
    networks:
      - server
    deploy:
      mode: replicated
      replicas: 1

  mongo:
    image: ${MONGO_IMAGE}
    # Se eliminan container_name y restart
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_INITDB_ROOT_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_INITDB_ROOT_PASSWORD}
    ports: # Se mantienen ports si son necesarios
      - "${MONGO_PORT}:27017"
    volumes:
      - mongo_data:/data/db
    networks:
      - server
    deploy:
      mode: replicated
      replicas: 1

volumes:
  sql_data:
  keycloak_data:
  mongo_data:

networks:
  server:
    driver: overlay
    attachable: true
  traefik-public:
    external: true
```
