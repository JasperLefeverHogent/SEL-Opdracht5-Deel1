# SEL: Opdracht 5 Docker

---

# 1. Installatie Docker

1. `sudo apt-get remove docker docker-engine docker.io containerd runc`
2. `sudo apt-get update`

```bash
#dependencies
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

#PGP key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

#add repos
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#install
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

#verify
sudo docker run hello-world
```

# 2. Installatie Vaultwarden

---

## 2.1 Docker Container

```bash
#Pullen container image en starten
sudo docker pull vaultwarden/server:latest
sudo docker run -d --name vaultwarden -v /vw-data/:/data/ -p 80:80 vaultwarden/server:latest
```

## 2.2 Firewall

```bash
sudo ufw allow 80 #HTTP
sudo ufw allow 443 #HTTPS
```

## 2.3 HTTPS

```bash
#Pullen container image en starten
sudo docker pull vaultwarden/server:latest
sudo docker run -d --name vaultwarden -v /vw-data/:/data/ -p 80:80 vaultwarden/server:latest

#SSL dir maken
sudo mkdir /ssl
sudo cd /ssl

#creating CA keys
sudo openssl genpkey -algorithm RSA -aes128 -out private-ca.key -outform PEM -pkeyopt rsa_keygen_bits:2048

#CA cert
sudo openssl req -x509 -new -nodes -sha256 -days 3650 -key private-ca.key -out self-signed-ca-cert.crt

#bitwarden key
sudo openssl genpkey -algorithm RSA -out bitwarden.key -outform PEM -pkeyopt rsa_keygen_bits:2048

#bitw cert
sudo openssl req -new -key bitwarden.key -out bitwarden.csr

#CREATE FILE bitwarden.ext
sudo nano bitwarden.ext

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = UWIPHIER
IP.2 = 127.0.0.1

#create btwarden cert signed by CA
sudo openssl x509 -req -in bitwarden.csr -CA self-signed-ca-cert.crt -CAkey private-ca.key -CAcreateserial -out bitwarden.crt -days 365 -sha256 -extfile bitwarden.ext

#Opstarten met de certificaten
sudo docker run --name vaultwarden -e ROCKET_TLS='{certs="/ssl/bitwarden.crt",key="/ssl/bitwarden.key"}' -v /ssl/:/ssl/ -v /vw-data/:/data/ -p 443:80 vaultwarden/server:latest
	# --env , -e Set environment variables
	# --volume , -v	Bind mount a volume
	# -p Publish a container's port(s) to the host

#heropstarten
sudo docker start vaultwarden
```

# 3. Installatie Portainer

---

```bash
#volume maken voor de container
sudo docker volume create portainer_data

#Opstarten van de container
sudo docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/files-portainer portainer/portainer-ce:2.11.1

#openen van de poort voor portainer
sudo ufw allow 9443 #portainer

```

- Inspecteer jouw containers: kan je de Portainer en Vaultwarden containers zien?
    - Ja op home en doorklikken naar containers
- Kan je de Vaultwarden afsluiten en terug opstarten via Portainer?
    - Ja met de bedieningsknoppen bovenaan

# 4. Installatie Docker Compose

---

```bash
#Download latest stable release
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose

#enable executable perm
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose

#check version
docker compose version
```

## 4.1 Werken met Docker compose

```bash
#bestand maken
sudo nano docker-compose.yml

#plak 
version: '3'
services:
    vaultwarden:
    image: vaultwarden/server:latest
	  container_name: vaultwarden
        ports:
            - 4123:80
        volumes:
            - ./.files-vaultwarden:/data

sudo docker compose up -d

**#BIJ FOUT met COMPOSER**
sudo usermod -aG docker ${USER}
su - ${USER}
sudo usermod -aG docker HIERUWUSERNAME
```

### Docker compose containers omzetten

```bash
#DOCKER 
version: '3'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    ports:
      - 443:80
    volumes:
      - ./.files-vaultwarden:/data
      - /ssl/:/ssl
    environment:
      - ROCKET_TLS={certs="/ssl/bitwarden.crt",key ="/ssl/bitwarden.key"}
      - ROCKET_PORT=80
    restart: always
  portainer:
    image: portainer/portainer-ce:2.11.1
    container_name: portainer
    ports:
      - 8000:8000
      - 9443:9443
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes/portainer_data/_data:/files-portainer

#In 1 keer containers met compose sluiten en verwijderen
docker compose down --rmi all -v
```

- Je maakt een enkel “docker-compose.yml”-bestand met daarin beide containers in (is dit verstandig? Waarom wel of niet?).
    - Je kan dit doen als je altijd beide containers nodig hebt
    
    docker compose -f /vaultwarden-portainer.yml up -d
    

# Evaluatie

Toon na afwerken het resultaat aan je begeleider. Criteria voor beoordeling:

- sudo docker --version
- sudo docker compose --version
- Je kan de command line instructies om een Vaultwarden container op te zetten toelichten.
- Je kan de command line instructies om een Portainer container op te zetten toelichten.
- Je kan een Vaultwarden container opzetten via Docker Compose op de command line. Je kan surfen naar en inloggen op deze container. Je hebt deze ook gekoppeld aan een client.
    - docker compose up -d
- Je kan een Portainer container opzetten via Docker Compose op de command line. Je kan surfen naar en inloggen op deze container. Portainer en Vaultwarden worden op het Portainer dashboard weergegeven met als status “Running”.
    - docker compose up -d