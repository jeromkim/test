MINIO
Nginx config

events {}
http {
  server {
    listen 80;
    listen [::]:80;
    location / {
       rewrite /console(/|$)(.*) /$2  break;
       proxy_pass         http://minio:9001/;
       proxy_redirect     off;
    }
  }
}
