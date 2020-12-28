Executive Summary
Objective
This document describes four deployments combining Kong and Aviatrix products:
Kuma Service Mesh Universal Multi Cloud deployment running on AWS and Azure.
Kong Enterprise running on AWS consuming a Microservice running on Azure.
Kong Enterprise running on AWS consuming a Canary Release deployed on two Azure regions
Kong for Kubernetes running on AWS EKS consuming a Microservice running on Azure.

All the communication between the components are based on Private IP only and abstracted by Aviatriax capabilities as Transit and Spoke Gateways.

The tutorial assumes you have already installed the following products
Kubectl
HTTPie and Curl
Jq



Aviatrix
Aviatrix Platform Installation
Follow installation instructions described in https://docs.aviatrix.com/StartUpGuides/aviatrix-cloud-controller-startup-guide.html
Scroll at the very bottom, "Aviatrix Secure Networking Platform - BYOL", to get redirected to its landing page:



Aviatrix CoPilot Installation
Follow the installation instructions described in https://docs.aviatrix.com/HowTos/copilot_overview.html
The Aviatrix CoPilot landing is here: https://aws.amazon.com/marketplace/pp/B086WNWGKX



Azure Account and Aviatrix App
Since we are going to deploy a AWS-Azure connection, create a Aviatrix Account on Azure following the steps described here: https://docs.aviatrix.com/HowTos/Aviatrix_Account_Azure.html
You should have an Aviatrix Application registered like this:

Aviatrix Controller and CoPilot Addresses
For this tutorial the addresses for both Controller and CoPilot are:
Controller: https://52.52.157.228
CoPilot: https://18.144.34.56

Aviatrix Transit and Spoke Gateways
To get started, create a Transit Gateway on both AWS and Azure. For each one of them instantiate and attach a Spoke Gateway. The topology for the two first deployments are the following. Notice we have created, initially, three VMs:
Aviatrix-Kuma-CP: Kuma Control Plane instance on AWS
Aviatrix-Source: Source Microservice instance on AWS
Aviatrix-Destination: Destination Microservice instance on Azure




VMs creation
AWS - Kuma Control Plane
Go to EC2 dashboard and click on "Launch Instance". Select "Ubuntu Server 18.04 LTS (HVM), and "t2.large" Instance Type.

Choose the default VPC, a Public Subnet and enable "Auto-assign Public IP".

Change the Security Group to enable access from any address.

The VM used in this lab has the following IPs defined:
Public IP: 54.193.209.29
Private IP: 172.31.18.74

All the topologies described on this document will use the Private IP.

$ ssh -i "acqua.pem" ubuntu@ec2-54-193-209-29.us-west-1.compute.amazonaws.com
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.3.0-1035-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Nov  2 21:09:25 UTC 2020

  System load:  0.0               Users logged in:                1
  Usage of /:   41.2% of 7.69GB   IP address for eth0:            172.31.18.74
  Memory usage: 6%                IP address for docker0:         172.17.0.1
  Swap usage:   0%                IP address for br-8b786d5492bc: 172.18.0.1
  Processes:    114

 * Introducing self-healing high availability clustering for MicroK8s!
   Super simple, hardened and opinionated Kubernetes for production.

     https://microk8s.io/high-availability

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

0 packages can be updated.
0 updates are security updates.

New release '20.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


*** System restart required ***
Last login: Mon Nov  2 19:33:05 2020 from 186.204.144.4



AWS - Source
Create another instance for the caller Microservice. Again, choose the default VPC, a Public Subnet and enable "Auto-assign Public IP".

Create a specific Security Group to enable access from any address.

The VM used in this lab has the following IPs defined:
Public IP: 18.144.34.124
Private IP: 172.31.30.79

All the topologies described on this document will use the Private IP.

$ ssh -i "acqua.pem" ubuntu@ec2-18-144-34-124.us-west-1.compute.amazonaws.com
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.3.0-1035-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Nov  7 12:00:30 UTC 2020

  System load:  1.2               Processes:           116
  Usage of /:   14.6% of 7.69GB   Users logged in:     0
  Memory usage: 2%                IP address for eth0: 172.31.30.79
  Swap usage:   0%


0 packages can be updated.
0 updates are security updates.

New release '20.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Sat Nov  7 12:00:27 2020 from 186.204.144.4
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.



Azure - Destination
Go to Azure portal and create another Ubuntu Server 18.04 VM.

The VM used in this lab has the following IPs defined:
Public IP: 40.76.224.126
Private IP: 10.0.0.5

All the topologies described on this document will use the Private IP.


$ ssh -i kong.pem kong@40.76.224.126
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.4.0-1031-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Nov  2 21:11:10 UTC 2020

  System load:  0.0                Processes:              123
  Usage of /:   10.3% of 28.90GB   Users logged in:        1
  Memory usage: 5%                 IP address for eth0:    10.0.0.5
  Swap usage:   0%                 IP address for docker0: 172.17.0.1

 * Introducing self-healing high availability clustering for MicroK8s!
   Super simple, hardened and opinionated Kubernetes for production.

     https://microk8s.io/high-availability

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

0 packages can be updated.
0 updates are security updates.

New release '20.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Mon Nov  2 18:50:15 2020 from 186.204.144.4




Kuma Service Mesh - 2-Microservice Application
Our Kuma Testing Environment is based on a simple Microservice-to-Microservice communication scenario. We have only two Microservices in this example developed in Python:
"benigno": provides a "hello" endpoint where it echoes the current datetime.
"magnanimo": calls "benigno" using the "hello" endpoint.



AWS - Microservice Source
ssh -i "acqua.pem" ubuntu@ec2-18-144-34-124.us-west-1.compute.amazonaws.com

Install utilities
sudo apt-get -y update
sudo apt -y install httpie
sudo apt -y install python3-pip
sudo pip3 install flask

Magnanimo Microservice
mkdir MS1
cd MS1

Inside "MS1" directory we create a "app.py" file

from flask import Flask
from datetime import datetime
import requests

app = Flask(__name__)

@app.route('/')
def index():
    return 'Hello World, Magnanimo', 200

@app.route('/sentence')
def sentence():
    return 'Hello World, Magnanimo - Kong', 200

@app.route('/hello')
def hello():
    return 'Hello World, Magnanimo: %s' % (datetime.now())

@app.route('/hw2')
def hw2():
    r = requests.get("http://benigno:5000/hello")
    return r.text

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=4000)


"magnanimo" will be listening to port 4000. Notice that it defines a couple of routes where it just returns simple strings. However, the route "hw2" is where we call the "benigno" Microservice, listening to port 5000.

Running Magnanimo
$ python3 app.py
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:4000/ (Press CTRL+C to quit)


In another terminal make your first request:
$ http :4000
HTTP/1.0 200 OK
Content-Length: 22
Content-Type: text/html; charset=utf-8
Date: Thu, 05 Nov 2020 12:41:20 GMT
Server: Werkzeug/1.0.1 Python/3.6.9

Hello World, Magnanimo

You can request the other route also:
$ http :4000/hello
HTTP/1.0 200 OK
Content-Length: 50
Content-Type: text/html; charset=utf-8
Date: Thu, 05 Nov 2020 12:41:35 GMT
Server: Werkzeug/1.0.1 Python/3.6.9

Hello World, Magnanimo: 2020-11-05 12:41:35.444341

However, if you request the "hw2" route, since you don't have our "benigno" Microservice running, you get an error:
$ http :4000/hw2
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Length: 290
Content-Type: text/html; charset=utf-8
Date: Thu, 05 Nov 2020 12:41:51 GMT
Server: Werkzeug/1.0.1 Python/3.6.9

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>



Azure
ssh -i kong.pem kong@40.76.224.126

Benigno Microservice
mkdir MS2
cd MS2

Inside "MS2" directory we create a "app.py" file

from flask import Flask
from datetime import datetime

app = Flask(__name__)


@app.route('/')
def index():
    return 'Hello World, Benigno', 200

@app.route('/sentence')
def sentence():
    return 'Hello World, Benigno - Kong', 200

@app.route('/hello')
def hello():
    return 'Hello World, Benigno: %s' % (datetime.now())

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)


Running Benigno
$ python3 app.py
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)


Open a third terminal to make your request. The same way we did with "magnanimo", we can exercise "benigno"'s routes:

$ http :5000/hello
HTTP/1.0 200 OK
Content-Length: 48
Content-Type: text/html; charset=utf-8
Date: Thu, 05 Nov 2020 12:43:15 GMT
Server: Werkzeug/1.0.1 Python/3.6.9

Hello World, Benigno: 2020-11-05 12:43:15.159587




Running Magnanimo-Benigno
In order to exercise the communication between the two Microservices we need to solve the "benigno" name used by "magnanimo"'s code. The simplest way to do it is to include an entry in your /etc/hosts file with the benigno's Private IP:
10.0.0.5	benigno

Now, if we send a request to "magnanimo"'s route "hw2", as we tried to do before, we receive a message coming from "benigno":

$ http :4000/hw2
HTTP/1.0 200 OK
Content-Length: 48
Content-Type: text/html; charset=utf-8
Date: Thu, 05 Nov 2020 12:44:16 GMT
Server: Werkzeug/1.0.1 Python/3.6.9

Hello World, Benigno: 2020-11-05 12:44:16.222321


Kuma Service Mesh
Kuma Download
Install Kuma on all three VMs with the following process:
mkdir kuma
cd kuma
curl -L https://kuma.io/installer.sh | sh -

Path Environment Variable
Create a script called kuma.sh with a line including your working directory like this:

export PATH=.:$HOME/kuma/kuma-0.7.3/bin:$PATH

Execute it like this
. ./kuma.sh

Checking the version
$ kumactl version
Kuma: 0.7.3

Kuma CP
Open a terminal on AWS VM:
ssh -i "acqua.pem" ubuntu@ec2-54-193-209-29.us-west-1.compute.amazonaws.com
. ./kuma.sh

Environment variables
Notice we're using the VM's Private IP:
unset KUMA_ADMIN_SERVER_PUBLIC_ENABLED
unset KUMA_GENERAL_ADVERTISED_HOSTNAME
unset KUMA_ADMIN_SERVER_PUBLIC_TLS_CERT_FILE
unset KUMA_ADMIN_SERVER_PUBLIC_TLS_KEY_FILE
unset KUMA_ADMIN_SERVER_PUBLIC_INTERFACE
unset KUMA_ADMIN_SERVER_PUBLIC_CLIENT_CERTS_DIR

export KUMA_ADMIN_SERVER_PUBLIC_ENABLED=true
export KUMA_GENERAL_ADVERTISED_HOSTNAME=172.31.18.74
export KUMA_ADMIN_SERVER_PUBLIC_TLS_CERT_FILE=/home/ubuntu/server-cert/kuma.pem
export KUMA_ADMIN_SERVER_PUBLIC_TLS_KEY_FILE=/home/ubuntu/server-cert/kuma.key
export KUMA_ADMIN_SERVER_PUBLIC_INTERFACE=172.31.18.74
export KUMA_ADMIN_SERVER_PUBLIC_CLIENT_CERTS_DIR=/home/ubuntu/client-cert


Server and Client Digital Certificates issuing
rm /home/ubuntu/server-cert/kuma.pem
yes | rm /home/ubuntu/server-cert/kuma.key

kumactl generate tls-certificate --cert-file=/home/ubuntu/server-cert/kuma.pem --key-file=/home/ubuntu/server-cert/kuma.key --type=server --cp-hostname=172.31.18.74


rm /home/ubuntu/client-cert/kuma-client.pem
yes | rm /home/ubuntu/client-cert/kuma-client.key

kumactl generate tls-certificate --cert-file=/home/ubuntu/client-cert/kuma-client.pem --key-file=/home/ubuntu/client-cert/kuma-client.key --type=client


Start the Control Plane
kuma-cp run






Kuma DP 1 - Azure
Open a terminal on Azure VM:
ssh -i kong.pem kong@40.76.224.126
. ./kuma.sh

Copy the client certificate and key
rm /home/kong/client-cert/kuma-client.pem
yes | rm /home/kong/client-cert/kuma-client.key

From your laptop copy the "kong.pem" file to AWS Kuma CP VM:
scp -i "acqua.pem" ../Azure/kong.pem ubuntu@ec2-54-193-209-29.us-west-1.compute.amazonaws.com:/home/ubuntu

From the AWS Kuma CP VM copy the client certificate and key to Azure VM:
scp -i "kong.pem" /home/ubuntu/client-cert/kuma-client.pem kong@40.76.224.126:/home/kong/client-cert/kuma-client.pem

scp -i "kong.pem" /home/ubuntu/client-cert/kuma-client.key kong@40.76.224.126:/home/kong/client-cert/kuma-client.key

Define the CP
Inside Azure VM, define the the CP running on AWS VM:
kumactl config control-planes list
kumactl config control-planes remove --name kuma-cp

kumactl config control-planes add \
  --name kuma-cp --address http://172.31.18.74:5681 \
  --admin-client-cert /home/kong/client-cert/kuma-client.pem \
  --admin-client-key /home/kong/client-cert/kuma-client.key


Define the Dataplane
Notice we're using Azure VM's Private IP:
kumactl delete dataplane benigno
kumactl get dataplanes


echo "type: Dataplane
mesh: default
name: benigno
networking:
  address: 10.0.0.5
  inbound:
  - port: 9000
    servicePort: 5000
    tags:
      kuma.io/service: benigno" | kumactl apply -f -


Generate the Dataplane Token
Since the Dataplane will be running on a different VM, the communication with the Control Plane should be based on tokens generated by the Control Plane:
rm /home/kong/kuma-dp-benigno-token

kumactl generate dataplane-token --dataplane=benigno > /home/kong/kuma-dp-benigno-token


Start the Dataplane
kuma-dp run \
  --name=benigno \
  --mesh=default \
  --cp-address=http://172.31.18.74:5681 \
  --dataplane-token-file=/home/kong/kuma-dp-benigno-token


Kuma DP 2 - AWS Microservice Source
Open a terminal on AWS Source VM:
ssh -i "acqua.pem" ubuntu@ec2-18-144-34-124.us-west-1.compute.amazonaws.com
. ./kuma.sh
mkdir client-cert

Copy the client certificate and key
rm /home/ubuntu/client-cert/kuma-client.pem
yes | rm /home/ubuntu/client-cert/kuma-client.key

From your laptop copy the "acqua.pem" file to AWS Kuma CP VM:
scp -i "acqua.pem" acqua.pem ubuntu@ec2-54-193-209-29.us-west-1.compute.amazonaws.com:/home/ubuntu


From the AWS Kuma CP VM copy the client certificate and key to Azure VM:
scp -i "acqua.pem" /home/ubuntu/client-cert/kuma-client.pem ubuntu@ec2-18-144-34-124.us-west-1.compute.amazonaws.com:/home/ubuntu/client-cert/kuma-client.pem

scp -i "acqua.pem" /home/ubuntu/client-cert/kuma-client.key ubuntu@ec2-18-144-34-124.us-west-1.compute.amazonaws.com:/home/ubuntu/client-cert/kuma-client.key


Define the CP
Notice we're using the same client certificate and key as Azure VM:
kumactl config control-planes list
kumactl config control-planes remove --name kuma-cp

kumactl config control-planes add \
  --name kuma-cp --address http://172.31.18.74:5681 \
  --admin-client-cert /home/ubuntu/client-cert/kuma-client.pem \
  --admin-client-key /home/ubuntu/client-cert/kuma-client.key


Define the Dataplane
kumactl delete dataplane magnanimo
kumactl get dataplanes

echo "type: Dataplane
mesh: default
name: magnanimo
networking:
  address: 172.31.30.79
  inbound:
  - port: 9000
    servicePort: 4000
    tags:
      kuma.io/service: magnanimo
      kuma.io/protocol: http
  outbound:
  - port: 5000
    tags:
      kuma.io/service: benigno" | kumactl apply -f -




Generate the Dataplane Token
rm /home/ubuntu/kuma-dp-magnanimo-token

kumactl generate dataplane-token --dataplane=magnanimo > /home/ubuntu/kuma-dp-magnanimo-token

Start the Dataplane
kuma-dp run \
  --name=magnanimo \
  --mesh=default \
  --cp-address=http://172.31.18.74:5681 \
  --dataplane-token-file=/home/ubuntu/kuma-dp-magnanimo-token


Checking Kuma console
Redirect your browser to AMS Kuma Control Plane VM using its Public IP: 54.193.209.29



Change Magnanimo to use loopback address
from flask import Flask
from datetime import datetime
import requests

app = Flask(__name__)

@app.route('/')
def index():
    return 'Hello World, Magnanimo', 200

@app.route('/sentence')
def sentence():
    return 'Hello World, Magnanimo - Kong', 200

@app.route('/hello')
def hello():
    return 'Hello World, Magnanimo: %s' % (datetime.now())

@app.route('/hw2')
def hw2():
    #r = requests.get("http://benigno:5000/hello")
    r = requests.get("http://localhost:5000/hello")
    return r.text

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=4000)


Testing Magnanimo-Benigno connection
Now we're going to test the connection with both sidecars included:
$ http 18.144.34.124:4000/hw2
HTTP/1.0 200 OK
Content-Length: 48
Content-Type: text/html; charset=utf-8
Date: Fri, 06 Nov 2020 19:31:36 GMT
Server: Werkzeug/1.0.1 Python/3.6.9

Hello World, Benigno: 2020-11-06 19:31:36.552909



Network connection problem
Now we're going to simulate a network problem. The easiest way for doing it is to remove the Outbound Rules out of the AWS VM's Security Group, so the AWS VM won't be able to connect to Azure VM anymore.
Start the loop first from your laptop:
while [ 1 ]; do curl http://18.144.34.124:4000/hw2;echo; done

Go to AWS console and remove the Outbound Rules from the Microservice Source VM:


The loop should stop when the deletion completes.
$ while [ 1 ]; do curl http://54.193.209.29:4000/hw2;echo; done
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>







Aviatrix FlightPath
Using FlightPath to check the error:

As we can see, FlightPath reports that there's no Security Group on the source to allow the Source VM to reach 10.0.0.5, the Private IP of the Destination Microservice VM.



Recreate the Outbound Rule
Recreate the Outbound Rule to allow the communication between the two VMs again:

And then see the loop getting back to normal:
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>

Hello World, Benigno: 2020-11-06 20:49:22.896943
Hello World, Benigno: 2020-11-06 20:49:23.591546
Hello World, Benigno: 2020-11-06 20:49:24.315930





Kong Enterprise - AWS -> Microservice - Azure
The following topology explores the Kong Enterprise API Gateway consuming Microservices running on different Cloud environments:

AWS Source
Go to the Source VM and install Kong Enterprise:
Install utilities
sudo apt-get -y update
sudo apt -y install httpie
sudo apt -y install jq

Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker.service
sudo systemctl start docker.service
sudo systemctl status docker.service

Check the installation with:
sudo docker info

If you want to stop Docker:
sudo systemctl stop docker.service

Portainer installation
sudo docker volume create portainer_data

sudo docker run --name portainer -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer

sudo docker start portainer

Using the EC2's public address, check the installation:
http://54.193.209.29:9000

At the first access, Portainer asks to define the admin's password. Choose "portainer".

After choosing the "Local - Manage the local Docker environment", we'll see its home page.

Docker login
sudo docker login -u cacquaviva -p xxx kong-docker-kong-enterprise-edition-docker.bintray.io


Docker image pull
sudo docker pull kong-docker-kong-enterprise-edition-docker.bintray.io/kong-enterprise-edition:2.1.4.1-alpine

docker tag c7d384e60fbc kong-ee


License environment variable setting
You have to use the license file provided by Kong.

export KONG_LICENSE_DATA='{"license":{"signature":"xxxxxx33b157036d5b6ebb79530a4109f2960b22ab9861be73270cf1d714cca045204a7d9b1be39d797c3213c76ae91e6c64262d870c4979d1c","payload":{"customer":"Kong_SE_Demo","license_creation_date":"2019-11-03","product_subscription":"Kong Enterprise Edition","admin_seats":"5","support_plan":"None","license_expiration_date":"2020-12-12","license_key":"xxxxxxxx"},"version":1}}'

Kong PostgreSQL installation
sudo docker run -d --name kong-ee-database \
   -p 5432:5432 \
   -e "POSTGRES_USER=kong" \
   -e "POSTGRES_DB=kong" \
   -e "POSTGRES_HOST_AUTH_METHOD=trust" \
   postgres:latest

sudo docker run --rm --link kong-ee-database:kong-ee-database \
   -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=kong-ee-database" \
   -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
   -e "KONG_PASSWORD=kong" \
   -e "POSTGRES_PASSWORD=kong" \
   kong-ee kong migrations bootstrap


sudo docker run -d --name kong-ee --link kong-ee-database:kong-ee-database \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-ee-database" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PORTAL_API_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_PORTAL_API_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
  -e "KONG_ADMIN_GUI_LISTEN=0.0.0.0:8002, 0.0.0.0:8445 ssl" \
  -e "KONG_PORTAL=on" \
  -e "KONG_PORTAL_GUI_PROTOCOL=http" \
  -e "KONG_PORTAL_GUI_HOST=localhost:8003" \
  -e "KONG_PORTAL_SESSION_CONF={\"cookie_name\": \"portal_session\", \"secret\": \"portal_secret\", \"storage\":\"kong\", \"cookie_secure\": false}" \
  -e "KONG_LICENSE_DATA=$KONG_LICENSE_DATA" \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 8444:8444 \
  -p 8002:8002 \
  -p 8445:8445 \
  -p 8003:8003 \
  -p 8446:8446 \
  -p 8004:8004 \
  -p 8447:8447 \
  kong-ee



Create a Docker network and connect Kong to it.
sudo docker network create kong-net
sudo docker network connect kong-net kong-ee
sudo docker network connect kong-net kong-ee-database

sudo docker stop kong-ee
sudo docker stop kong-ee-database

sudo docker start kong-ee-database
sudo docker start kong-ee


Test the Control Plane
Test the installation pointing your laptop browser to http:18.144.34.124:8002 to open Kong Manager or use Httpie

http :8001 | jq .version


Azure Destination
Go to Azure Destination VM and install Docker and Portainer just like you did on AWS Source VM.

Start the Microservice which will be consumed by Kong Enterprise

$ sudo docker run --name benigno -h benigno -d -p 5000:5000 claudioacquaviva/benigno

In order to allow the communication between the containers we have to use the --link option.

We can see our containers now

$ sudo docker container ls
CONTAINER ID  IMAGE                       COMMAND             CREATED         STATUS         PORTS                   NAMES
7b958468aadc  claudioacquaviva/magnanimo  "python app.py"     2 seconds ago   Up 1 second    0.0.0.0:4000->4000/tcp  magnanimo



The same "http" command issued before has to work this time:
$ http :5000/hello
HTTP/1.0 200 OK
Content-Length: 66
Content-Type: text/html; charset=utf-8
Date: Fri, 19 Apr 2019 16:23:31 GMT
Server: Werkzeug/0.15.2 Python/3.6.2

Hello World, Benigno: 2019-04-19 16:23:31.661124





AWS - Kong Service & Route
Go to AWS Source VM to define Kong Enterprise Route and Service. Notice we're using Azure VM's Private IP.

http delete :8001/routes/benignoroute
http delete :8001/services/benignoservice

http :8001/services name=benignoservice url='http://10.0.0.5:5000'
http :8001/services/benignoservice/routes name='benignoroute' paths:='["/benigno"]'


$ http :8000/benigno/hello
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 48
Content-Type: text/html; charset=utf-8
Date: Tue, 27 Oct 2020 17:37:44 GMT
Server: Werkzeug/1.0.1 Python/3.8.3
Via: kong/2.1.4.0-enterprise-edition
X-Kong-Proxy-Latency: 0
X-Kong-Upstream-Latency: 135

Hello World, Benigno: 2020-10-27 17:37:44.402514


From your laptop start a loop:

while [ 1 ]; do curl http://18.144.34.124:8000/benigno/hello;echo; done


Network connection problem
Now we're going to simulate a network problem. This time, we're going to update the Inbound port rule on Azure. Start a loop, so Kong Enterprise can consume the Microservice:
while [ 1 ]; do curl http://18.144.34.124:8000/benigno/hello;echo; done

Go to FlightPath to check the current settings:


FlightPath show there's a Security Group at Destination allowing all communications from all sources:


Change the Inbound Port Rule denying all communication:







Run FlightPath again to see the changes. And then that the loop we started before stops. Change the Inbound rule again to allow the communication between Kong Enterprise and the Microservice.

Kong Enterprise - Canary & Upstream - AWS ->  2 Microservices - Azure
The following topology evolves the previous scenario implementing a Canary Release with Kong Enterprise API Gateway Upstreams consuming Microservices running on different Azure Regions environments:


Azure Destination 2
Create another Azure Destination VM to run a Canary version of the Microservice.




Go to Azure Destination VM and install Docker and Portainer just like you did on AWS Source VM.

The VM used in this lab has the following IPs defined:
Public IP: 20.55.233.136
Private IP: 10.0.6.5

$ ssh -i kong.pem kong@20.55.233.136
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.4.0-1031-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Nov  7 18:12:59 UTC 2020

  System load:  0.6               Processes:           131
  Usage of /:   4.5% of 28.90GB   Users logged in:     0
  Memory usage: 2%                IP address for eth0: 10.0.6.5
  Swap usage:   0%


0 packages can be updated.
0 updates are security updates.

New release '20.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Sat Nov  7 18:12:56 2020 from 186.204.144.4
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.



Start the Microservice which will be consumed by Kong Enterprise

$ sudo docker run --name benigno_rc -h benigno_rc -d -p 5000:5000 claudioacquaviva/benigno_rc


We can see our containers now

$ sudo docker container ls
CONTAINER ID  IMAGE                       COMMAND             CREATED         STATUS         PORTS                   NAMES
7b958468aadc  claudioacquaviva/magnanimo  "python app.py"     2 seconds ago   Up 1 second    0.0.0.0:4000->4000/tcp  magnanimo



Test the Microservice

$ http :5000/hello
HTTP/1.0 200 OK
Content-Length: 64
Content-Type: text/html; charset=utf-8
Date: Sat, 07 Nov 2020 17:50:58 GMT
Server: Werkzeug/1.0.1 Python/3.8.3

Hello World, Benigno, Canary Release: 2020-11-07 17:50:58.599000



Upstreams
Go to AWS Source VM to define the Kong Upstream and Targets

http delete :8001/routes/benignoroute
http delete :8001/services/benignoservice

http :8001/upstreams/benignoupstream/targets

http delete :8001/upstreams/benignoupstream/targets/676ce3e3-61af-4af7-8f0b-b6dcd8669e79

http delete :8001/upstreams/benignoupstream/targets/b7ea6ea5-2946-4d8b-b853-2e0fc1f08daf

http delete :8001/upstreams/benignoupstream


Notice we're using only Private IPs to define the targets of the Upstream

curl http://localhost:8001/upstreams -H "Content-Type: application/json" --data '{ "name": "benignoupstream", "healthchecks": { "active": { "unhealthy": { "timeouts": 5, "interval": 3 }, "healthy": { "interval": 3 } } } }'


http :8001/upstreams/benignoupstream/targets target='10.0.0.5:5000' weight:=80 

http :8001/upstreams/benignoupstream/targets target='10.0.6.5:5000' weight:=20




Kong Service & Route
http :8001/services name=benignoservice url='http://benignoupstream'
http :8001/services/benignoservice/routes name='benignoroute' paths:='["/benigno"]'


Check the Upstream using Kong Manager




Testing the Canary
From your laptop start a loop

$ while [ 1 ]; do curl http://18.144.34.124:8000/benigno/hello;echo; done
Hello World, Benigno: 2020-11-07 18:26:06.719219
Hello World, Benigno: 2020-11-07 18:26:07.421007
Hello World, Benigno: 2020-11-07 18:26:08.133473
Hello World, Benigno: 2020-11-07 18:26:08.829361
Hello World, Benigno: 2020-11-07 18:26:09.531435
Hello World, Benigno, Canary Release: 2020-11-07 18:26:10.225378
Hello World, Benigno: 2020-11-07 18:26:10.922018
Hello World, Benigno: 2020-11-07 18:26:11.623978
Hello World, Benigno: 2020-11-07 18:26:12.327982
Hello World, Benigno: 2020-11-07 18:26:13.033339
Hello World, Benigno: 2020-11-07 18:26:13.731275
Hello World, Benigno: 2020-11-07 18:26:14.428957
Hello World, Benigno, Canary Release: 2020-11-07 18:26:15.118639
Hello World, Benigno: 2020-11-07 18:26:15.810771

Network connection problem
First of all, go to AWS Source VM to check Kong Enterprise log. As you can see, the Upstream is being consuming with successfully:
$ sudo docker logs -f kong-ee 
...
186.204.144.4 - - [07/Nov/2020:19:42:55 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:42:56 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:42:57 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:42:57 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:42:58 +0000] "GET /benigno/hello HTTP/1.1" 200 64 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:42:59 +0000] "GET /benigno/hello HTTP/1.1" 200 64 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:00 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:00 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:01 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:02 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:02 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:03 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:04 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:04 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:05 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:06 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"
186.204.144.4 - - [07/Nov/2020:19:43:07 +0000] "GET /benigno/hello HTTP/1.1" 200 48 "-" "curl/7.64.1"

Now, to Azure console and change the Inbound port rule for the Destination 1:


Eventually, Kong will report a communication errors with the target:
$ sudo docker logs -f kong-ee | grep timeout
...
2020/11/07 20:12:57 [warn] 25#0: *229762 [lua] healthcheck.lua:1104: log(): [healthcheck] (0a9ff2c5-f883-4267-b684-4ea54d890bf0:benignoupstream) unhealthy TIMEOUT increment (1/5) for '10.0.0.5(10.0.0.5:5000)', context: ngx.timer
2020/11/07 20:13:03 [warn] 24#0: *229868 [lua] healthcheck.lua:1104: log(): [healthcheck] (0a9ff2c5-f883-4267-b684-4ea54d890bf0:benignoupstream) unhealthy TIMEOUT increment (2/5) for '10.0.0.5(10.0.0.5:5000)', context: ngx.timer, client: 186.204.144.4, server: 0.0.0.0:8001
2020/11/07 20:13:06 [warn] 24#0: *229922 [lua] healthcheck.lua:1104: log(): [healthcheck] (0a9ff2c5-f883-4267-b684-4ea54d890bf0:benignoupstream) unhealthy TIMEOUT increment (3/5) for '10.0.0.5(10.0.0.5:5000)', context: ngx.timer, client: 186.204.144.4, server: 0.0.0.0:8001
2020/11/07 20:13:09 [warn] 24#0: *229970 [lua] healthcheck.lua:1104: log(): [healthcheck] (0a9ff2c5-f883-4267-b684-4ea54d890bf0:benignoupstream) unhealthy TIMEOUT increment (4/5) for '10.0.0.5(10.0.0.5:5000)', context: ngx.timer, client: 186.204.144.4, server: 0.0.0.0:8001
2020/11/07 20:13:12 [warn] 25#0: *230028 [lua] healthcheck.lua:1104: log(): [healthcheck] (0a9ff2c5-f883-4267-b684-4ea54d890bf0:benignoupstream) unhealthy TIMEOUT increment (5/5) for '10.0.0.5(10.0.0.5:5000)', context: ngx.timer
2020/11/07 20:13:55 [error] 25#0: *229738 upstream timed out (110: Operation timed out) while connecting to upstream, client: 186.204.144.4, server: kong, request: "GET /benigno/hello HTTP/1.1", upstream: "http://10.0.0.5:5000/hello", host: "18.144.34.124:8000"


And then Kong Enterprise will considered the target as "unhealthy" and all the traffic will be routed to the Canary release:
Hello World, Benigno: 2020-11-07 20:12:30.407419
Hello World, Benigno: 2020-11-07 20:12:31.109394
Hello World, Benigno: 2020-11-07 20:12:31.803286
Hello World, Benigno: 2020-11-07 20:12:32.491483
Hello World, Benigno: 2020-11-07 20:12:33.192559
Hello World, Benigno: 2020-11-07 20:12:33.907276
Hello World, Benigno: 2020-11-07 20:12:34.611988
Hello World, Benigno: 2020-11-07 20:12:35.304506
Hello World, Benigno: 2020-11-07 20:12:36.403015
Hello World, Benigno: 2020-11-07 20:12:37.099648
Hello World, Benigno, Canary Release: 2020-11-07 20:12:37.800617
Hello World, Benigno: 2020-11-07 20:12:38.490777
Hello World, Benigno: 2020-11-07 20:12:39.187686
Hello World, Benigno, Canary Release: 2020-11-07 20:12:39.881459
Hello World, Benigno: 2020-11-07 20:12:40.571679
Hello World, Benigno: 2020-11-07 20:12:41.274995
Hello World, Benigno: 2020-11-07 20:12:41.969210
Hello World, Benigno: 2020-11-07 20:12:42.670314
Hello World, Benigno: 2020-11-07 20:12:43.370595
Hello World, Benigno, Canary Release: 2020-11-07 20:12:44.069548
Hello World, Benigno: 2020-11-07 20:12:44.753727
Hello World, Benigno: 2020-11-07 20:12:45.452768
Hello World, Benigno: 2020-11-07 20:12:46.151474
Hello World, Benigno: 2020-11-07 20:12:46.846144
Hello World, Benigno: 2020-11-07 20:12:47.553717
Hello World, Benigno: 2020-11-07 20:12:48.255586
Hello World, Benigno, Canary Release: 2020-11-07 20:12:48.954555
Hello World, Benigno: 2020-11-07 20:12:49.653569
Hello World, Benigno: 2020-11-07 20:12:50.360089
Hello World, Benigno: 2020-11-07 20:12:51.050730
Hello World, Benigno: 2020-11-07 20:12:51.745728
Hello World, Benigno: 2020-11-07 20:12:52.441256
Hello World, Benigno: 2020-11-07 20:12:53.144320
Hello World, Benigno, Canary Release: 2020-11-07 20:12:53.841462
Hello World, Benigno: 2020-11-07 20:12:54.537677
Hello World, Benigno, Canary Release: 2020-11-07 20:13:55.234548
Hello World, Benigno, Canary Release: 2020-11-07 20:13:55.931111
Hello World, Benigno, Canary Release: 2020-11-07 20:13:56.628942
Hello World, Benigno, Canary Release: 2020-11-07 20:13:57.322353
Hello World, Benigno, Canary Release: 2020-11-07 20:13:58.009775
Hello World, Benigno, Canary Release: 2020-11-07 20:13:58.748093
Hello World, Benigno, Canary Release: 2020-11-07 20:13:59.434772
Hello World, Benigno, Canary Release: 2020-11-07 20:14:00.124639
Hello World, Benigno, Canary Release: 2020-11-07 20:14:00.815915
Hello World, Benigno, Canary Release: 2020-11-07 20:14:01.506058


Change Inbound port rule for the Destination 1 again allowing the communication. Go to Kong Manager to mark the target as "healthy" again, or send a post request like this:
http post :8001/upstreams/benignoupstream/targets/09f1a2e6-1238-4dcf-980d-993452fb8613/healthy


Kong Enterprise starts consuming the target again:
Hello World, Benigno, Canary Release: 2020-11-08 15:56:33.697963
Hello World, Benigno, Canary Release: 2020-11-08 15:56:34.384813
Hello World, Benigno, Canary Release: 2020-11-08 15:56:35.074784
Hello World, Benigno, Canary Release: 2020-11-08 15:56:35.759504
Hello World, Benigno, Canary Release: 2020-11-08 15:56:36.442890
Hello World, Benigno, Canary Release: 2020-11-08 15:56:37.125920
Hello World, Benigno: 2020-11-08 15:56:40.839762
Hello World, Benigno, Canary Release: 2020-11-08 15:56:41.523427
Hello World, Benigno, Canary Release: 2020-11-08 15:56:42.209416
Hello World, Benigno: 2020-11-08 15:56:42.899521
Hello World, Benigno: 2020-11-08 15:56:43.593825
Hello World, Benigno: 2020-11-08 15:56:44.288826
Hello World, Benigno: 2020-11-08 15:56:45.385573
Hello World, Benigno: 2020-11-08 15:56:46.072313
Hello World, Benigno: 2020-11-08 15:56:46.764580
Hello World, Benigno: 2020-11-08 15:56:47.464463
Hello World, Benigno: 2020-11-08 15:56:48.166014
Hello World, Benigno: 2020-11-08 15:56:48.870017
Hello World, Benigno: 2020-11-08 15:56:49.566855
Hello World, Benigno: 2020-11-08 15:56:50.273741
Hello World, Benigno: 2020-11-08 15:56:50.961764
Hello World, Benigno: 2020-11-08 15:56:51.651022
Hello World, Benigno, Canary Release: 2020-11-08 15:56:52.343054
Hello World, Benigno: 2020-11-08 15:56:53.029080
Hello World, Benigno, Canary Release: 2020-11-08 15:56:53.722638
Hello World, Benigno, Canary Release: 2020-11-08 15:56:54.408615
Hello World, Benigno: 2020-11-08 15:56:55.097550
Hello World, Benigno: 2020-11-08 15:56:55.801263
Hello World, Benigno, Canary Release: 2020-11-08 15:56:56.492920
Hello World, Benigno: 2020-11-08 15:56:57.181569
Hello World, Benigno, Canary Release: 2020-11-08 15:56:57.878695
Hello World, Benigno: 2020-11-08 15:56:58.571057
Hello World, Benigno: 2020-11-08 15:56:59.274867
Hello World, Benigno, Canary Release: 2020-11-08 15:56:59.966679
Hello World, Benigno: 2020-11-08 15:57:00.652761
Hello World, Benigno: 2020-11-08 15:57:01.354019
Hello World, Benigno: 2020-11-08 15:57:02.051621






K4K8S - AWS -> Microservice - Azure
This topology is similar to the one describing Kong Enterprise and a Microservice. The difference is that we use Kong for Kubernetes on an EKS Cluster:


AWS - EKS Cluster
Creating the EKS Cluster
eksctl creates implicitly a specific VPC for our EKS Cluster.

eksctl create cluster --name kong-aviatrix --version 1.18 --nodegroup-name standard-workers --node-type t3.medium --nodes 1

Deleting the EKS Clusters
In case you need to delete it, run the following command:

eksctl delete cluster --name kong-aviatrix


Deploying Magnanimo
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: magnanimo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: magnanimo
  template:
    metadata:
      labels:
        app: magnanimo
    spec:
      containers:
      - name: magnanimo
        image: claudioacquaviva/magnanimo
        ports:
        - containerPort: 4000
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: magnanimo
  labels:
    app: magnanimo
spec:
  type: ClusterIP
  ports:
  - port: 4000
    name: http
  selector:
    app: magnanimo
EOF


kubectl port-forward service/magnanimo 4000:4000

$ http :4000
HTTP/1.0 200 OK
Content-Length: 22
Content-Type: text/html; charset=utf-8
Date: Tue, 15 Sep 2020 15:02:01 GMT
Server: Werkzeug/1.0.1 Python/3.8.3

Hello World, Magnanimo



Deploying Benigno as a local Kubernetes Service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: benigno-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: benigno
  template:
    metadata:
      labels:
        app: benigno
        version: v1
    spec:
      containers:
      - name: benigno
        image: claudioacquaviva/benigno
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: benigno
  labels:
    app: benigno
spec:
  type: ClusterIP
  ports:
    - port: 5000
      name: http
  selector:
    app: benigno
EOF


$ kubectl get service --all-namespaces
NAMESPACE     NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
default       benigno      ClusterIP   10.100.195.99   <none>        5000/TCP        22m
default       kubernetes   ClusterIP   10.100.0.1      <none>        443/TCP         60m
default       magnanimo    ClusterIP   10.100.117.99   <none>        4000/TCP        35m
kube-system   kube-dns     ClusterIP   10.100.0.10     <none>        53/UDP,53/TCP   60m



$ http :4000/hw2
HTTP/1.0 200 OK
Content-Length: 48
Content-Type: text/html; charset=utf-8
Date: Mon, 02 Nov 2020 15:40:10 GMT
Server: Werkzeug/1.0.1 Python/3.8.3

Hello World, Benigno: 2020-11-02 15:40:10.255001


Undeploy Benigno
kubectl delete service benigno
kubectl delete deployment benigno-v1


Define External Benigno Service running on Azure
cat <<EOF | kubectl apply -f -
kind: Service
apiVersion: v1
metadata:
  name: benigno
spec:
  ports:
    -
      name: benigno
      protocol: TCP
      port: 5000
      targetPort: 5000
---
kind: Endpoints
apiVersion: v1
metadata:
  name: benigno
subsets:
  -
    addresses:
      -
        ip: 10.0.0.5
    ports:
      -
        port: 5000
        name: benigno
EOF


kubectl delete service benigno
kubectl delete endpoint benigno



Links

https://community.aviatrix.com/t/q6hh0wb/building-public-cloud-networks-and-troubleshooting-connectivity-in-aws-azure-gcp-oci-video
https://docs.aviatrix.com/HowTos/transitvpc_faq.html
https://aviatrix.com/learn-center/answered-multi-cloud/how-to-do-multicloud-networking-abstraction-and-orchestration-across-aws-azure-and-google/


https://docs.aviatrix.com/HowTos/transitvpc_faq.html
https://docs.aviatrix.com/HowTos/gateway.html?highlight=gateway
https://docs.aviatrix.com/HowTos/transitvpc_workflow.html

https://docs.aviatrix.com/HowTos/Aviatrix_Account_Azure.html#azure-arm
https://docs.aviatrix.com/HowTos/GettingStartedAzureToAWSAndGCP.html





