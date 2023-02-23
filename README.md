# Not READY

# Easy-to-way-to-install-elastic-kibana-with-terraform
Easy way to install elastic search and kibana for adicional look guides "how to use install apm server with fleet server"


## Spanish Versión

La idea de este tutorial es demostrar como crear con un cluster con la cantidad de nodos que tu quieras de elastic search, un kibana sobre una cloud como digital ocean, esto es aplicable tambien para otras nubes cambiando los providers, aún asi te invito y te recomiendo usar elastic cloud https://cloud.elastic.co/home 

### Configuracion de iniciales variables



```terraform 
variable "cluster-name" {
  type = string
  description = "The name of the cluster"
}

variable "cluster-elastic-nodes" {
  type = number
  description = "How many nodes to create for the cluster"
}

```


### Proveedor de digital ocean



```terraform 
terraform {
  required_providers {
    digitalocean = {
      source = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
}
provider "digitalocean" {
  token = "<your Digital Ocean token>"
}

data "digitalocean_ssh_key" "ssh_key" {
  name = "<your ssh key name>"
}

```


### Definición de instalacias

Lo primero que haremos sera definir las maquinas que ocuparemos para correr en nuestra cluster elastic, la cual
ocupa la variable cluster-elastic-nodes la al ejecutarse nos preguntara cuantas maquinas queremos usar

```terraform 
resource "digitalocean_droplet" "elastic" {
  count  = var.cluster-elastic-nodes
  image  = "ubuntu-22-04-x64"
  name   = "${var.cluster-name}-elastic-search-${count.index+1}"
  region = "sfo3"
  size   = "s-2vcpu-2gb"
  ssh_keys = [data.digitalocean_ssh_key.ssh_key.id]
}
```

Lo segundo sera definir la máquina en donde se instalar el visualizador grafico de kibana

```terraform 
resource "digitalocean_droplet" "kibana" {
  count  = 1
  image  = "ubuntu-22-04-x64"
  name   = "${var.cluster-name}-kibana-${count.index+1}"
  region = "sfo3"
  size   = "s-2vcpu-2gb"
  ssh_keys = [data.digitalocean_ssh_key.ssh_key.id]
}
```



### Instalacion de elastic search

```terraform
resource "null_resource" "install_elasticsearch" {
  count = var.cluster-elastic-nodes

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.elastic[count.index].ipv4_address
  }
  provisioner "remote-exec" {
    inline = [
      "curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -",
      "echo 'deb https://artifacts.elastic.co/packages/8.x/apt stable main' | sudo tee /etc/apt/sources.list.d/elastic-8.x.list",
      "sudo apt update",
      "sudo apt install -y elasticsearch",
      "echo 'network.host: 0.0.0.0' >> /etc/elasticsearch/elasticsearch.yml",
      "echo 'cluster.name: ${var.cluster-name}' >> /etc/elasticsearch/elasticsearch.yml"
    ]
  }
}```


```terraform
resource "null_resource" "configurate_elasticsearch_master" {
  count = 1
  depends_on = [
        null_resource.install_elasticsearch
        ]
  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.elastic[count.index].ipv4_address
  }
  provisioner "remote-exec" {
    inline = [
      "sudo systemctl daemon-reload",
      "sudo systemctl enable elasticsearch.service",
      "sudo systemctl start elasticsearch.service",
      "sleep 10",
      "/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana > /tmp/kibana-token.json",
      "/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node > /tmp/node-token.json",
      "echo y | /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -s > /tmp/password.txt"
    ]
  }
  provisioner "local-exec" {
    command = "ssh-keyscan -H ${digitalocean_droplet.elastic[count.index].ipv4_address}>> ~/.ssh/known_hosts && scp root@${digitalocean_droplet.elastic[count.index].ipv4_address}:/tmp/kibana-token.json . && scp root@${digitalocean_droplet.elastic[count.index].ipv4_address}:/tmp/node-token.json . && scp root@${digitalocean_droplet.elastic[count.index].ipv4_address}:/tmp/password.txt ."
  }
}


```terraform
resource "null_resource" "enable_elasticsearch" {
    depends_on = [
        null_resource.configurate_elasticsearch_master
    ]
  count = "${var.cluster-elastic-nodes - 1}"

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = "${digitalocean_droplet.elastic[count.index+1].ipv4_address}"
  }
  provisioner "file" {
    source      = "node-token.json"
    destination = "/tmp/token.json"
  }

  provisioner "remote-exec" {
    inline = [
      "token=`cat /tmp/token.json` && echo y | /usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token $token",
      "sudo systemctl daemon-reload",
      "sudo systemctl enable elasticsearch.service",
      "sudo systemctl start elasticsearch.service"
    ]
  }
}


```


### Instalación de kibana

```terraform
resource "null_resource" "install_kibana" {
  depends_on = [
    null_resource.configurate_elasticsearch_master
  ]
  count = 1
  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.kibana[count.index].ipv4_address
  }
  provisioner "file" {
    source      = "kibana-token.json"
    destination = "/tmp/token.json"
  }
  provisioner "remote-exec" {
    inline = [
      "curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -",
      "echo 'deb https://artifacts.elastic.co/packages/8.x/apt stable main' | sudo tee /etc/apt/sources.list.d/elastic-8.x.list",
      "sudo apt update",
      "sudo apt install -y kibana",
      "sudo sed -i 's/#server.host: \"localhost\"/server.host: \"0.0.0.0\"/' /etc/kibana/kibana.yml",
      "echo 'elasticsearch.hosts: [\"${join("\",\"", digitalocean_droplet.elastic.*.ipv4_address)}\"]' >> /etc/kibana/kibana.yml",
      "sudo systemctl enable kibana",
      "sudo systemctl start kibana",
      "token=`cat /tmp/token.json` && /usr/share/kibana/bin/kibana-setup --enrollment-token $token"

    ]
  }
}
```

### Informacion para conectarse

```terraform 
output "Kibana_URL" {
    depends_on = [
      null_resource.configurate_elasticsearch_master,
        null_resource.install_kibana
    ]
  value       = "wait 2 minute to connect to http://${digitalocean_droplet.kibana[0].ipv4_address}:5601"
  description = "Now you can connect to kibana with this url:"
}

output "Password" {
    depends_on = [
      null_resource.configurate_elasticsearch_master,
        null_resource.install_kibana
    ]
  value       = "Your elastic password can found in password.txt file"
  description = "Password elastic is: "
}
````


### Conclusión


Estamos listo para puedas ocupar elastic search en tu propia infraestructura ahora te invitamos a que puedas continuar utilizandolo en tus propios proyectos.

Te recomendamos que veas los siguientes proyectos:

1.  [Como agregar observabilidad de elastic search a nuestro proyecto de next js](https://github.com/elastic-community/apm-elastic-next.js)
1.  [La mejor forma de generar observabilidad y monitoreo a nuestras aplicaciones en la nube ](https://github.com/elastic-community/ATP-elastic-APM)
1.  [Monitoreo de aplicaciones en una aplicación Golang con Elastic APM.](https://github.com/elastic-community/Golang-application-with-Elastic-APM)
