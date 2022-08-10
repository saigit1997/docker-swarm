# ansible-docker-swarm 




Setup Docker Swarm with Ansible.

In this setup I have a client node, which will be my jump box, as it will be used to ssh with the docker user to my swarm nodes with passwordless ssh access.

## Pre-Check

Hosts file: 

```
$ cat /etc/hosts
10.0.8.2 client
192.168.1.10 swarm-manager
192.168.1.11 swarm-worker-1
192.168.1.12 swarm-worker-2
```

SSH Config:

```
$ cat ~/.ssh/config 
Host swarm-manager
  Hostname swarm-manager
  User centos
  IdentityFile /tmp/key.pem
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null

Host swarm-worker-1
  Hostname swarm-worker-1
  User centos
  IdentityFile /tmp/key.pem
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null

Host swarm-worker-2
  Hostname swarm-worker-2
  User centos
  IdentityFile /tmp/key.pem
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

Install Ansible:

```
$ sudo sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
$ sudo sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
$ sudo dnf update
$ sudo dnf install epel-release
$ sudo yum install python3-pip
$ sudo pip3 install ansible

```

Ensure passwordless ssh is working:

```
$ ansible -i inventory.ini -u centos -m ping all
client | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
swarm-manager | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
swarm-worker-2 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
swarm-worker-1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

## Deploy Docker Swarm

```
$ ansible-playbook -i inventory.ini -u centos deploy-swarm.yml 
PLAY RECAP 
   
swarm-manager              : ok=18   changed=4    unreachable=0    failed=0   
swarm-worker-1             : ok=15   changed=1    unreachable=0    failed=0   
swarm-worker-2             : ok=15   changed=1    unreachable=0    failed=0   
```

SSH to the Swarm Manager and List the Nodes:

```
$ sudo docker node ls
ID                            HOSTNAME                                       STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
cfgyuvspsjlt2968ixrd0o9ng     ip-172-31-5-205.ap-south-1.compute.internal    Ready     Active                          20.10.17
tmq0j7mopnw01p7rbin1a2uwq     ip-172-31-8-63.ap-south-1.compute.internal     Ready     Active                          20.10.17
lem5wmtbkgqbsgw32g56okcf6 *   ip-172-31-14-229.ap-south-1.compute.internal   Ready     Active         Leader           20.10.17
```

Deploy nginx stack with 3 replicas of nginx service using ansible:

```
$ ansible-playbook -i inventory.ini -u centos nginx.yml
$ docker service ls
 sudo docker service ls
ID             NAME        MODE         REPLICAS   IMAGE        PORTS
3nm12vru9x9n   myservice   replicated   3/3        nginx:1.13   *:80->80/tcp

$ docker service ps nginx
             
sudo docker service ps myservice
ID             NAME              IMAGE        NODE                                           DESIRED STATE   CURRENT STATE             ERROR     PORTS
sduqchjeqzn8   myservice.1       nginx:1.13   ip-172-31-8-63.ap-south-1.compute.internal     Running         Running 13 minutes ago              *:80->80/tcp,*:80->80/tcp
odnsdl8uu9v0   myservice.2       nginx:1.13   ip-172-31-5-205.ap-south-1.compute.internal    Running         Running 13 minutes ago              *:80->80/tcp,*:80->80/tcp
llwt85gwauo5   myservice.3       nginx:1.13   ip-172-31-14-229.ap-south-1.compute.internal   Running         Running 3 minutes ago               *:80->80/tcp,*:80->80/tcp                       
   
```

Test the Application:

```
$  curl -i 3.110.183.249
HTTP/1.1 200 OK
Server: nginx/1.13.12
Date: Wed, 10 Aug 2022 10:12:14 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Mon, 09 Apr 2018 16:01:09 GMT
Connection: keep-alive
ETag: "5acb8e45-264"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
