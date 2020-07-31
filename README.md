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
aws ec2 import-key-pair --key-name "ApiKurniawanKeyPair" --public-key-material fileb://~/.ssh/api-kurniawan-dev.pub
```
Add the key name (*ApiKurniawanKeyPair*) to `ec2.params` and deploy

```shell script
aws cloudformation deploy \
  --stack-name "ku-api" \
  --template-file "api.yml" \
  --parameter-overrides $(cat "api.params" | tr '\n' ' ') \
  --capabilities CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset
```
When this is done, an EC2 with public elastic IP will be accessible via `api.kurniawan.dev` port `443`, `80`, and `22`.

## Certbot

Then SSH `ssh ec2-user@api.kurniawan.dev`

Follow [certbot installation guide](https://certbot.eff.org/lets-encrypt/centosrhel7-nginx). Which is basically:

```shell script
# install certbot
sudo yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum -y install certbot python2-certbot-nginx

# install certbot_dns_route53
sudo yum -y install python-pip
certbot --version
pip install certbot_dns_route53==<certbot_version>
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

If successful, we should get something like
> IMPORTANT NOTES:
> Congratulations! Your certificate and chain have been saved at:
      /home/ec2-user/letsencrypt/config/live/api.kurniawan.dev/fullchain.pem
      Your key file has been saved at:
      /home/ec2-user/letsencrypt/config/live/api.kurniawan.dev/privkey.pem
      Your cert will expire on 2020-10-11. To obtain a new or tweaked
      version of this certificate in the future, simply run certbot
      again. To non-interactively renew *all* of your certificates, run
      "certbot renew"
    - Your account credentials have been saved in your Certbot
      configuration directory at /home/ec2-user/letsencrypt/config. You
      should make a secure backup of this folder now. This configuration
      directory will also contain certificates and private keys obtained
      by Certbot so making regular backups of this folder is ideal.


Add cron job for certificate renewal
```
echo "0 4 * * * ec2-user certbot renew --dns-route53 --logs-dir /home/ec2-user/letsencrypt/logs/ --config-dir /home/ec2-user/letsencrypt/config/ --work-dir /home/ec2-user/letsencrypt/work/ --non-interactive --post-hook \"docker exec ku-webserver nginx -s reload\"" | sudo tee -a /etc/crontab > /dev/null
```

## Docker

1. [Install docker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html)
2. Renew ssh session to make sure `ec2-user` has been added to the docker group
3. [Install docker compose](https://docs.docker.com/compose/install/)


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
