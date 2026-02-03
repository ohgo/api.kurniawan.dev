# api.kurniawan.dev
Reverse proxy for api.kurniawan.dev

To minimize cost the infrastructure for [api.kurniawan.dev](https://api.kurniawan.dev) consists of a single EC2 instance.
A dockerized nginx with letsencrypt certificate is running as reverse proxy.

## EC2
Create SSH KeyPair
```
ssh-keygen -t rsa -b 4096 -C "ec2-user@api.kurniawan.dev" -f ~/.ssh/api-kurniawan-dev
```
Add to `~/.ssh/config`
```
Host api.kurniawan.dev
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/api-kurniawan-dev
```
Import to AWS
```
aws ec2 import-key-pair --key-name "ApiKurniawanKP" --public-key-material fileb://~/.ssh/api-kurniawan-dev.pub
```
Add the key name (*ApiKurniawanKP*) to `api.params` and deploy

```shell script
aws cloudformation deploy \
  --stack-name "ku-api" \
  --template-file "api.yml" \
  --parameter-overrides $(cat "api.params" | tr '\n' ' ') \
  --capabilities CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset
```
When this is done, an EC2 be accessible via `api.kurniawan.dev` port `443`, `80`, and `22`.

## Certbot

Make sure SSHLocation in `api.params` is correct. Then SSH `ssh ec2-user@api.kurniawan.dev`

Follow [certbot installation guide](https://certbot.eff.org/instructions). Which should be something like

```shell script
# install python
sudo yum install -y python3 python3-pip
pip3 install --upgrade pip

# install certbot_dns_route53
sudo pip3 install certbot certbot-dns-route53 boto3

# verify dns-route53 plugin is installed
sudo /usr/local/bin/certbot plugins
```

Generate certificate
```shell script
mkdir ~/letsencrypt && \
mkdir ~/letsencrypt/config && \
mkdir ~/letsencrypt/work && \
mkdir ~/letsencrypt/logs && \
certbot certonly --dns-route53 -d api.kurniawan.dev \
  --config-dir ~/letsencrypt/config \
  --work-dir ~/letsencrypt/work \
  --logs-dir ~/letsencrypt/logs \
  -m aries@kurniawan.dev \
  --agree-tos \
  --non-interactive
```

Add cron job for certificate renewal
```
echo "0 4 * * * ec2-user certbot renew --dns-route53 --logs-dir /home/ec2-user/letsencrypt/logs/ --config-dir /home/ec2-user/letsencrypt/config/ --work-dir /home/ec2-user/letsencrypt/work/ --non-interactive --post-hook \"docker exec ku-webserver nginx -s reload\"" | sudo tee -a /etc/crontab > /dev/null
```

## Docker

1. [Install docker and compose in ec2](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-container-image.html)


## Run application
Prepare `.env` file
```shell script
KU_PROFILE_TAG=latest
SPOTIFY_TOKEN_PASSWORD=<SPOTIFY_TOKEN_PASSWORD>
SPOTIFY_TOKEN_REFRESH=<SPOTIFY_TOKEN_REFRESH>
GOODREADS_KEY=<GOODREADS_KEY>
```
Upload files to EC2
```shell script
scp -r nginx ec2-user@api.kurniawan.dev:~
scp docker-compose.yml ec2-user@api.kurniawan.dev:~
scp .env ec2-user@api.kurniawan.dev:~
```

Run docker compose
```shell script
docker-compose up -d
```
