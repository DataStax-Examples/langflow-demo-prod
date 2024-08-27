# Load Balancing and Routing

Langflow (at least at version 1.0.15) expects all URLs to be rooted from `/`: there is not a straightforward way to configure a `base_url` or similar.
This is a challenge for load balancers and other application routers. This goal of this page is to inform you about some tricks you can use
to configure your own system; it is based on the popular Nginx framework but should be easily translatable to whatever framework you use.

## Server Proxy

Consider this Nginx `server` configuration, which allows Langflow developers to access the UI at `langflow.dev.internal` on port `80`,
whilst routing requests to `langflow.dev.service` on port `7860`:

```
    server {
        listen 80;

        server_name langflow.dev.internal;

        location / {
            proxy_pass http://langflow.dev.service:7860;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
```

## Server Proxy with Routing

To add an application routing based on URL (in this case `/chat`), use the route to identify the destination location, but
remove the route from the URL before passing to Langflow:

```
    server {
        listen 80;

        server_name langflow.test.internal;

        location /chat {
            rewrite ^/chat/(.*)$ /$1 break;
            proxy_pass http://langflow.test.service:7860;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
```

Two important things to note about this routing approach:

1. The Langflow frontend will not work without a '/' location, but backend API calls should. You can combine both locations within the `server` configuration.
2. Some returned content (such as when an API is called with `stream=true`) includes URLs that will not be prefixed (i.e. `stream_url` will begin `/api/v1/...`); you will need to add the prefix within your application logic.

