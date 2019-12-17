# Nginx reverse proxy

A reverse proxy is like a middleman (proxy) between a user (client) making a request to that proxy
and that proxy making requests and retrieving its results from other servers.

![](https://www.pvsm.ru/images/2019/06/18/odin-iz-sotni-sposobov-publikacii-neskolkih-production-proektov-na-odnom-servere.png))

### 1. Nginx in docker

Nginx reverse proxy is based on the official nginx image.
For nginx to interact with the necessary services, they need to be on the same docker network. 
And in order for nginx to be accessible from outside, you must open the port.

docker-compose.yml:

```
nginx-reverse-proxy:
    image: nginx:1.17.6
    container_name: nginx-reverse-proxy
    networks:
      - united-network
    ports:
      - 80:80
```

### 2. Nginx.conf

Here is the simplest nginx file configuration:

```
events {                        #required
  worker_connections  1024;     #part
}
http {
  server {
      listen          80;
      server_name     example.voicesell.com;

      location / {
        proxy_pass  http://grafana:3000/;
      }
  }
}
```

In this configuration, nginx listens on port 80, all requests coming to the `example.voicesell.com` domain are redirected
to a web application that is running on `http://grafana:3000` (grafana is the name of the container of dockerized application).

Here is the configuration of the nginx file with basic authentication:

```
events {                        #required
  worker_connections  1024;     #part
}
http {
  server {
      listen          80;
      server_name     kibana.voicesell.com;

      location / {
        proxy_pass  http://kibana:5601/;
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd_kibana;
      }
  }
}
```

Inside a location that you are going to protect, specify the `auth_basic` directive and give a name to the password-protected area.
The name of the area will be shown in the username/password dialog window when asking for credentials.
Specify the `auth_basic_user_file` directive with a path to the .htpasswd file that contain user/password pairs.

The file with the usernames and encrypted passwords looks like this:
```
user1:$apr1$/woC1jnP$KAh0SsVn5qeSMjTtn0E9Q0
user2:$apr1$QdR8fNLT$vbCEEzDj7LyqCMyNpSoBh/
```

### 3. Using nginx configuration files in docker

To use the created configuration files in the nginx container, you must transfer it _(before starting the container)_
to the container using the volumes:
```
volumes:
  - ./nginx/nginx.conf:/etc/nginx/nginx.conf
  - ./nginx/.htpasswd_kibana:/etc/nginx/.htpasswd_kibana
```

### Complete nginx reverse proxy service in docker-compose.yml:
```
 nginx-reverse-proxy:
    image: nginx:1.17.6
    container_name: nginx-reverse-proxy
    networks:
      - united-network
    ports:
      - 80:80
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/.htpasswd_kibana:/etc/nginx/.htpasswd_kibana
      - ./nginx/.htpasswd_prometheus:/etc/nginx/.htpasswd_prometheus
```