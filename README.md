# DCC
Digital credential template for J-WEL repository

-----------------------------------------------------------------
Hi J-WEL Team,

Thanks for getting together with us today! Sharing some things that should be useful going forward:
•	recording of admin dashboard demo (https://drive.google.com/file/d/10iOoHBdM9DgT7VqUoh9jkXlErQmuz7DC/view?usp=sharing) - as James described, we have microservices that can be used separately with or without the dashboard but it seems like using the dashboard would be a good option for the types of credentials J-WEL wants to issue. Kerri should clarify in this video what goes into the credential template vs. the CSV file 
•	deployment guide (https://github.com/digitalcredentials/docs/blob/main/deployment-guide/DCCDeploymentGuide.md) - technical instructions for installing the issuer software 

-------------------------------------------------------------

Create the AWS Instance
Log in at https://aws.mit.edu/login 


Go to EC2 > Instances, click Launch Instances to create a new instance
name: VerifierPlus (name doesn't really matter)
instance size: t3.xlarge, 30 GB disk
security group: launch-wizard-1 (it's the "allow all traffic" one)
select an SSH key (you’ll likely choose to generate one, unless you want to use one you’d set for some other AWS instance, but you’d need the corresponding private key) 
Create instance.

Wait a few seconds while it spins up. 


SSH into the instance, specifying the location where you saved the key that you generated, and substituting your new IP:

ssh -i ~/Documents/keys/dz-verifier-plus-instance.pem ubuntu@3.87.83.9

Then:

sudo apt update
sudo apt upgrade
sudo reboot
Add SSH Keys


Add public keys for whoever needs login access (note: edit with ‘emacs’, 'nano' or 'vi' as you prefer):

vi ~/.ssh/authorized_keys

And paste in public keys, e.g.,


ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5soJCc2lBugf1VwiQnjjaTFA2FtsYR6XUcybKp6Py4HfKoJgope9loGLnz+FgJmDbGF+y5CjYRMsE1CsG7svTaMru9xO//Vq2jbyf62Yh+qrem+mOZgi6pjYOCg6owXV50bPYmmTAUNyY+UuJaBEW7txE6RgKCzKXWeZd+geBvDtD5JzJrmVxL/YX8wFJ1qSbwGiU1J2gQi29xeBSyfC0rcJ9G1TR65IhsWe/PRFYC8VqRT4Tsa/ei4PKt0Aikf9j6/fYOTmlkwuewQ3oD/ZSnvuqx5+gN0O6mqx1Zye2BdhBOxw0V+sW810kq9XItnEGLPVT4/0HekQMiMEuyEx1 someone@someone.net

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFyvEl+j9YS5w5SLwme3xwR/U4vcrnMs/+ZguA+Ed4d9 jc.chartrand@gmail.com

And save.

Install Node.js


sudo apt install build-essential

Install NVM (node version manager):


curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash

Install the LTS version of node:

nvm install --lts
Install Docker Engine
(from https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository )

sudo apt install \
    ca-certificates \
    curl \
    gnupg \
    
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y

sudo groupadd docker
sudo usermod -aG docker $USER
log out, log back in


sudo systemctl enable docker.service
sudo systemctl enable containerd.service

Test that Docker is running:

sudo docker run hello-world

---------------------------------------------
name: dashboard
services:
  coordinator:
    image: digitalcredentials/workflow-coordinator:0.1.0
    container_name: "ad-coordinator"
    environment:
      - ENABLE_STATUS_SERVICE=false
      - PUBLIC_EXCHANGE_HOST=https://${HOST}/api
      - TENANT_TOKEN_LEF_TEST=UNPROTECTED
    ports:
      - "4005:4005"
  signing:
    image: digitalcredentials/signing-service:0.2.0
    container_name: "ad-signing"
    environment:
      - TENANT_SEED_LEF_TEST=z1AeiPT496wWmo9BG2QYXeTusgFSZPNG3T9wNeTtjrQ3rCB
  transactions:
    image: digitalcredentials/transaction-service:0.1.1
    container_name: "ad-transactions"
  payload:
    image: digitalcredentials/dashboard:0.1.1host
    container_name: "ad-payload"
    depends_on: 
      - coordinator
      - redis
    environment:
      - COORDINATOR_URL=http://coordinator:4005
      - REDIS_URL=redis
      - REDIS_PORT=6379
      - MONGODB_URI=mongodb+srv://j-wel:DCCdatabase!@atlascluster.fmfmi.mongodb.net/
      - PAYLOAD_SECRET=aaaaaaaaaaaaaaaaaaaaaaaa
      - TENANT_NAME=lef_test
      - SMTP_HOST=mail.smtp2go.com
      - SMTP_USER=J-WEL
      - SMTP_PASS=ZlfU21NMShHz7P3o
      - EMAIL_FROM=J-WEL Digital Credentials<j-wel@mit.edu>
      - CLAIM_PAGE_URL=https://${HOST}/claim
      - PAYLOAD_PUBLIC_SERVER_URL=https://${HOST}
      - VIRTUAL_HOST=${HOST}
      - VIRTUAL_PATH=/
      - VIRTUAL_PORT=3000
      - LETSENCRYPT_HOST=${HOST}
    ports:
      - "3000:3000"
  claim-page:
    image: digitalcredentials/admin-dashboard-claim-page:1.0.0
    container_name: "ad-claim-page"
    environment:
      - VIRTUAL_HOST=${HOST}
      - VIRTUAL_PATH=/claim
      - VIRTUAL_DEST=/
      - VIRTUAL_PORT=8080
      - PUBLIC_PAYLOAD_URL=https://${HOST}/api
      - LETSENCRYPT_HOST=${HOST}
    depends_on: 
      - payload
    ports:
      - "8080:8080"
  redis:
    image: redis:alpine
    container_name: ad-redis
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
    ports:
      - "6379:6379"
  nginx-proxy:
    image: nginxproxy/nginx-proxy
    container_name: "ad-nginx"
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - html:/usr/share/nginx/html
      - certs:/etc/nginx/certs
      - vhost:/etc/nginx/vhost.d
  nginx-proxy-acme:
    image: nginxproxy/acme-companion
    container_name: "ad-nginx-proxy"
    volumes_from:
      - nginx-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - acme:/etc/acme.sh
    environment:
      - DEFAULT_EMAIL=j-wel@mit.edu
volumes:
  transactions:
  mongo_data:
  vhost:
  html:
  certs:
  acme:


