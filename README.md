# hostinger-configuration-microservices-with-docker
hostinger configuration microservices with docker
✅ 1. Instalacja Dockera

Jeśli nie masz jeszcze Dockera:

sudo apt update

sudo apt install -y docker.io docker-compose

sudo systemctl enable docker --now

✅ 2. Uruchomienie prywatnego Docker Registry

Docker ma gotowy oficjalny obraz registry:2.
Najprościej:

docker run -d \
  --name registry \
  -p 5000:5000 \
  -v /opt/registry/data:/var/lib/registry \
  registry:2


Twoje registry działa teraz na:

http://IP_VPS:5000


Bez HTTPS Docker zgłasza „insecure registry”, więc trzeba dodać wyjątek lub skonfigurować HTTPS (zalecane).

✅ 3. Dodanie uwierzytelniania (Basic Auth)

Utwórz folder na konfigurację:

mkdir -p /opt/registry/{auth,data}


Zainstaluj htpasswd:

sudo apt install apache2-utils


Dodaj użytkownika:

htpasswd -Bc /opt/registry/auth/htpasswd user1

✅ 4.1 Konfiguracja DOMENY NA HOSTINGER
  zmienić w 
  HOSTINGEER VPS => MANAGER DNS => EDYTUJ DNS => Zarządzaj rekordami DNS 
  ustawiasz lub zmieniasz jak juz jest:
  A	@	0	ADDERES IP SERVERA VPS	1440 

✅ 4.2 Konfiguracja Registry z Basic Auth + HTTPS (NGINX + Let's Encrypt)
4.1 Instalacja nginx
sudo apt install nginx

4.2 Uzyskanie certyfikatu Let’s Encrypt

(Użyj swojej domeny — np. registry.moje-domenka.pl)

sudo snap install core
sudo snap refresh core
sudo snap install certbot --classic
sudo certbot certonly --nginx -d registry.twojadomena.pl


Certyfikaty znajdą się w:

/etc/letsencrypt/live/registry.twojadomena.pl/

https://roadmap.sh/guides/setup-and-auto-renew-ssl-certificates

4.3 Konfiguracja NGINX jako reverse proxy

Plik:

/etc/nginx/sites-available/registry

  server {
      listen 80;
      server_name registry.twojadomena.pl;
      return 301 https://$host$request_uri;
  }
  
  server {
      listen 443 ssl;
      server_name registry.twojadomena.pl;
      
      ssl_certificate /etc/letsencrypt/live/registry.twojadomena.pl/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/registry.twojadomena.pl/privkey.pem;
      
      client_max_body_size 0;
      
      location / {
          root   /var/www/html/;
          index index.nginx-debian.html;
      }
  
      location /v2/ {
          proxy_pass http://127.0.0.1:5000;
          proxy_set_header Host $http_host;
  
          auth_basic "Docker Registry";
          auth_basic_user_file /opt/registry/auth/htpasswd;
      }
  }
  
Włącz i restartuj:

sudo ln -s /etc/nginx/sites-available/registry /etc/nginx/sites-enabled/registry

sudo nginx -t

sudo systemctl reload nginx

✅ 5. Uruchom registry z Basic Auth

Usuń poprzedni kontener i uruchom:

docker stop registry && docker rm registry

docker run -d \
  --name registry \
  -v /opt/registry/data:/var/lib/registry \
  -p 127.0.0.1:5000:5000 \
  registry:2

✅ 6. Logowanie i push do prywatnego repo

Logowanie:

docker login registry.twojadomena.pl


Tagowanie:

docker tag alpine:latest registry.twojadomena.pl/alpine:latest


Push:

docker push registry.twojadomena.pl/alpine:latest
