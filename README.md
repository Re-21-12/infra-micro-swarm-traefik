# infra-micro-swarm-traefik
```
version: '3.9'

services:
  portainer:
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:rw 
      - portainer_data:/data
    networks:
      - traefik-public-attachable
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      labels:
        - traefik.enable=true
        # ✅ Configuración estable para Portainer (HTTP interno)
        - traefik.http.services.portainer.loadbalancer.server.port=9000 
        - traefik.http.services.portainer.loadbalancer.server.scheme=http
        
        # 💡 MIDDLEWARE: Asegura el trailing slash (Portainer lo necesita)
        - traefik.http.middlewares.portainer-redirect.redirectregex.regex=^(.*)/$$
        - traefik.http.middlewares.portainer-redirect.redirectregex.replacement=$$1/

        # 1. ROUTER HTTP (Redirección a HTTPS)
        - traefik.http.routers.portainer-web.rule=Host(`portainer.corazondeseda.lat`)
        - traefik.http.routers.portainer-web.entrypoints=web
        
        # 2. ROUTER HTTPS 
        - traefik.http.routers.portainer-secure.rule=Host(`portainer.corazondeseda.lat`)
        - traefik.http.routers.portainer-secure.entrypoints=websecure
        - traefik.http.routers.portainer-secure.tls=true
        - traefik.http.routers.portainer-secure.tls.certresolver=le
        - traefik.http.routers.portainer-secure.service=portainer
        - traefik.http.routers.portainer-secure.middlewares=portainer-redirect@docker

  ## 🛡️ SERVICIO TRAEFIK (PROXY INVERSO Y SSL)
  traefik:
    image: traefik:v2.10
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=traefik-public-attachable
      - --providers.docker.swarmMode=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --entrypoints.webdashboard.address=:8081
      - --certificatesresolvers.le.acme.email=victora.caas@gmail.com
      - --certificatesresolvers.le.acme.storage=/etc/traefik/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
    ports:
      - "80:80" 
      - "443:443" 
      - "8081:8081"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_data:/etc/traefik 
    networks:
      - traefik-public-attachable
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      labels:
        - traefik.enable=true 
        - traefik.http.services.api.loadbalancer.server.port=8080 
        
        # Dashboard de Traefik - Acceso seguro y Path Stripping
        - traefik.http.routers.dashboard.rule=Host(`traefik.corazondeseda.lat`) # ⬅️ REGLA SIMPLIFICADA
        - traefik.http.routers.dashboard.entrypoints=websecure
        - traefik.http.routers.dashboard.service=api@internal
        - traefik.http.routers.dashboard.tls=true
        - traefik.http.routers.dashboard.tls.certresolver=le

volumes:
  portainer_data: {}
  traefik_data: {} 

networks:
  traefik-public-attachable:
    driver: overlay
    external: true
```
