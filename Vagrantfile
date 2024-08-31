# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure("2") do |config|

  if Vagrant.has_plugin? "vagrant-vbguest"
    config.vbguest.no_install  = true
    config.vbguest.auto_update = false
    config.vbguest.no_remote   = true
  end

  config.vm.define :balanceador do |balanceador|
    balanceador.vm.box = "bento/ubuntu-22.04"
    balanceador.vm.network :private_network, ip: "192.168.110.10"
    balanceador.vm.hostname = "balanceador"
    balanceador.vm.provision "shell1", type: "shell", run: "once", inline: <<-SHELL1
      apt update && apt upgrade -y
      apt install haproxy -y

      systemctl enable haproxy

      wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
      echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
      sudo apt install consul -y

      # Configurar Consul como servidor del clúster
      echo '{
        "datacenter": "dc1",
        "node_name": "haproxy",
        "data_dir": "/var/consul",
        "bind_addr": "192.168.110.10",
        "client_addr": "0.0.0.0",
        "server": true,
        "bootstrap_expect": 1
      }' | sudo tee /etc/consul.d/server.json

      echo 'HTTP/1.0 503 Service Unavailable
Content-Type: text/html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Servicio No Disponible</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
        }
    </style>
</head>
<body>
    <h1>Servicio No Disponible</h1>
    <p>Lo sentimos, pero el servicio no esta disponible en este momento. Intentalo de nuevo mas tarde.</p>
</body>
</html>' | sudo tee /etc/haproxy/errors/error.html

      echo "defaults
   mode http
   timeout connect 5s
   timeout client 1m
   timeout server 1m
   errorfile 503 /etc/haproxy/errors/error.html

frontend stats
   bind *:1936
   stats uri /
   stats show-legends
   no log

frontend http_front
   bind *:80
   default_backend http_back

backend http_back
    balance roundrobin
    server-template mywebapp 1-10 _web._tcp.service.consul resolvers consul    resolve-opts allow-dup-ip resolve-prefer ipv4 check

resolvers consul
    nameserver consul 127.0.0.1:8600
    accepted_payload_size 8192
    hold valid 5s" > /etc/haproxy/haproxy.cfg

    SHELL1

    balanceador.vm.provision "shell2", type: "shell", run: "always", inline: <<-SHELL2
      systemctl enable haproxy  
      nohup sudo consul agent -ui -config-dir=/etc/consul.d -bind=192.168.110.10 > /var/log/consul.log 2>&1 &
      systemctl restart haproxy
    SHELL2
  end

  config.vm.define :server1 do |server1|
    server1.vm.box = "bento/ubuntu-22.04"
    server1.vm.network :private_network, ip: "192.168.110.11"
    server1.vm.hostname = "server1"
    server1.vm.provision "shell1", type: "shell", run: "once", inline: <<-SHELL1
      wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
      echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
      apt update && apt upgrade -y
      sudo apt install consul -y
      echo '{
  "datacenter": "dc1",
  "node_name": "server1",
  "server": false,
  "data_dir": "/var/consul",
  "bind_addr": "192.168.110.11",
  "client_addr": "0.0.0.0",
  "retry_join": ["192.168.110.10"],
  "ui_config": {
    "enabled": true
  }
}' | sudo tee /etc/consul.d/agent.json
      sudo apt install nodejs -y
      sudo apt install npm -y
      mkdir -p /home/vagrant/consulService/app
      cd /home/vagrant/consulService/app
      npm init -y
      npm install consul -y
      npm install express -y
      echo 'const Consul = require("consul");
const express = require("express");

const SERVICE_NAME = "web";
const SERVICE_ID = "m" + process.argv[2];
const SCHEME = "http";
const HOST = "192.168.110.11";
const PORT = process.argv[2] * 1;
const PID = process.pid;

/* Inicializacion del server */
const app = express();
const consul = new Consul();

app.get("/health", function (req, res) {
    console.log("Health check!");
    res.end("Ok.");
});

app.get("/", (req, res) => {
  res.send(`<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Servidor web 1</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
        }
    </style>
</head>
<body>
    <h1>Servidor web 1</h1>
    <p>Hola desde el Servidor web 1</p>
</body>
</html>`);
});

app.listen(PORT, function () {
    console.log("Servicio iniciado en:" + SCHEME + "://" + HOST + ":" + PORT + "!");
});

/* Registro del servicio */
var check = {
  id: SERVICE_ID,
  name: SERVICE_NAME,
  address: HOST,
  port: PORT,
  check: {
    http: SCHEME + "://" + HOST + ":" + PORT + "/health",
    ttl: "5s",
    interval: "5s",
    timeout: "5s",
    deregistercriticalserviceafter: "1m"
  }
};

consul.agent.service.register(check, function(err) {
    if (err) throw err;
});' | sudo tee /home/vagrant/consulService/app/index.js
    SHELL1

    server1.vm.provision "shell2", type: "shell", run: "always", inline: <<-SHELL2
      nohup sudo consul agent -config-dir=/etc/consul.d -bind=192.168.110.11 > /var/log/consul.log 2>&1 &
      echo "Esperando a que Consul esté disponible..."
      until nc -z 192.168.110.11 8500; do
        sleep 5
      done
      nohup sudo node /home/vagrant/consulService/app/index.js 3000 > /home/vagrant/consulService/app/node.log 2>&1 &
    SHELL2
  end

  config.vm.define :server2 do |server2|
    server2.vm.box = "bento/ubuntu-22.04"
    server2.vm.network :private_network, ip: "192.168.110.12"
    server2.vm.hostname = "server2"
    server2.vm.provision "shell1", type: "shell", run: "once", inline: <<-SHELL1
      wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
      echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
      apt update && apt upgrade -y
      sudo apt install consul -y
      echo '{
  "datacenter": "dc1",
  "node_name": "server2",
  "server": false,
  "data_dir": "/var/consul",
  "bind_addr": "192.168.110.12",
  "client_addr": "0.0.0.0",
  "retry_join": ["192.168.110.10"],
  "ui_config": {
    "enabled": true
  }
}' | sudo tee /etc/consul.d/agent.json
      sudo apt install nodejs -y
      sudo apt install npm -y
      mkdir -p /home/vagrant/consulService/app
      cd /home/vagrant/consulService/app
      npm init -y
      npm install consul -y
      npm install express -y
      echo 'const Consul = require("consul");
const express = require("express");

const SERVICE_NAME = "web";
const SERVICE_ID = "m" + process.argv[2];
const SCHEME = "http";
const HOST = "192.168.110.12";
const PORT = process.argv[2] * 1;
const PID = process.pid;

/* Inicializacion del server */
const app = express();
const consul = new Consul();

app.get("/health", function (req, res) {
    console.log("Health check!");
    res.end("Ok.");
});

app.get("/", (req, res) => {
  res.send(`<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Servidor web 2</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
        }
        h1 {
            color: #333;
        }
        p {
            color: #666;
        }
    </style>
</head>
<body>
    <h1>Servidor web 2</h1>
    <p>Hola desde el Servidor web 2</p>
</body>
</html>`);
});

app.listen(PORT, function () {
    console.log("Servicio iniciado en:" + SCHEME + "://" + HOST + ":" + PORT + "!");
});

/* Registro del servicio */
var check = {
  id: SERVICE_ID,
  name: SERVICE_NAME,
  address: HOST,
  port: PORT,
  check: {
    http: SCHEME + "://" + HOST + ":" + PORT + "/health",
    ttl: "5s",
    interval: "5s",
    timeout: "5s",
    deregistercriticalserviceafter: "1m"
  }
};

consul.agent.service.register(check, function(err) {
    if (err) throw err;
    });' | sudo tee /home/vagrant/consulService/app/index.js
    SHELL1

    server2.vm.provision "shell2", type: "shell", run: "always", inline: <<-SHELL2
      nohup sudo consul agent -config-dir=/etc/consul.d -bind=192.168.110.12 > /var/log/consul.log 2>&1 &
      # Esperar a que Consul esté disponible
      echo "Esperando a que Consul esté disponible..."
      until nc -z 192.168.110.12 8500; do
        sleep 5
      done
      nohup sudo node /home/vagrant/consulService/app/index.js 3000 > /home/vagrant/consulService/app/node.log 2>&1 &
    SHELL2
  end

end
