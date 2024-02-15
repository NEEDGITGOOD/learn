#### Change

Change the "portainer.example.com" to your domain name!
````yaml
      - "traefik.http.routers.frontend.rule=Host(`portainer.example.com`)"
````

#### Usage