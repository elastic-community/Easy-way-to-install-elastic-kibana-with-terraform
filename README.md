
# Easy-to-way-to-install-elastic-kibana-with-terraform
Easy way to install elastic search and kibana for adicional look guides "how to use install apm server with fleet server" this post is available in english and spanish

## English Version
The idea of this tutorial is to demonstrate how to create a cluster with as many nodes as you want of Elasticsearch, along with a Kibana on a cloud such as Digital Ocean. This is applicable to other clouds by changing providers, but I still invite and recommend that you use Elastic Cloud https://cloud.elastic.co/home.


### Install with Terraform
To install Elastic with Kibana on our infrastructure, it is as simple as cloning this repository and running the following commands:
```bash
terraform init
terraform apply
```
We're ready to use our own Elasticsearch now. To find out what happened, I invite you to keep reading this guide.

### Initial variable configuration
We need two variables that will be prompted when executed, which are how many nodes we need and the name of our cluster
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

###  Digital Ocean Provider 
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

### Instance definition
To begin, we must define the machines we will use to run our Elastic cluster. This variable, called "cluster-elastic-nodes", will prompt us to specify the number of nodes we want, which can be from 1 to the number of machines we want to use. In this case, we will install it on an Ubuntu 22.04 machine with 2 vCPUs and 2 GB of RAM, but you can customize these requirements.
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
We need to define an instance where our Kibana will run, which is a data visualization tool and a graphical interface to access our Elasticsearch data. In this case, we will also use an Ubuntu 22.04 machine with 2 vCPUs and 2 GB of RAM, which you can customize according to your needs.
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



### Elasticsearch installation.
On all nodes of our cluster, we must perform a basic installation of Elasticsearch. This includes registering the necessary repositories, installing Elasticsearch, and configuring the cluster name.
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
}
```
Now, on the first node, we need to enable and start the Elasticsearch service, generate an enrollment token for the other nodes and for Kibana. We'll also change the password and export it to a text file, which you can download to your computer. Remember that this process only needs to be done on the first node.

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
```
If we have configured more than one node, we need to execute the following steps to upload the enrollment token and reconfigure the node to join the cluster mode.
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
### Kibana installation.
Great! We have already configured our Elasticsearch cluster. Now it's time to configure our Kibana, the data visualization tool. For this, we must configure the necessary repositories, install Kibana, and once it is started, enroll the system with our Elasticsearch cluster.

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
### Important information
After finishing the configuration, the URL of Kibana will be displayed so you can connect and start using the tool. Remember that the password will also be provided in the password.txt file of your cluster so you can access without any problem

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
  value       = "Your elastic password is ${file('password.txt')} can found in password.txt file"
  description = "Password elastic is: "
}
````


### Conclusion
Congratulations, you have installed Elasticsearch + Kibana on your infrastructure so that you can enjoy it in your projects. We invite you to continue using it in your own projects.

We recommend that you check out the following projects:


## Spanish Versión
La idea de este tutorial es demostrar como crear con un cluster con la cantidad de nodos que tu quieras de elastic search, un kibana sobre una cloud como digital ocean, esto es aplicable tambien para otras nubes cambiando los providers, aún asi te invito y te recomiendo usar elastic cloud https://cloud.elastic.co/home 

### Ejecuccion con terraform
Para instalar elastic con kibana nuestras infraestructura es tan simple como clonar este repositorio y ejecutar los siguiente comandos:
```bash
terraform init
terraform apply
```
Listo ya podemos utilizar nuestro propio elastic, para saber que fue lo que paso invito a continuar leyendo esta guia.

### Configuracion de iniciales variables
Necesitamos dos variable las cuales nos preguntara cuando se ejecuten la cuales son cuantos nodos necesitamos y el nombre de nuestro cluster.
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

### Definición de las instancias
Para empezar, debemos definir las máquinas que utilizaremos para ejecutar nuestra cluster elastic. Esta variable, llamada "cluster-elastic-nodes", nos pedirá que indiquemos la cantidad de nodos que deseamos, que puede ser desde 1 hasta el número de máquinas que queramos utilizar. En este caso, lo instalaremos en una máquina Ubuntu 22.04 con 2 vCPUs y 2 GB de RAM, pero puedes personalizar estos requisitos a tu gusto.
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
Además, necesitamos definir una instancia en la que se ejecutará nuestro Kibana, el cual es una herramienta de visualización de datos y una interfaz gráfica para acceder a nuestros datos de Elasticsearch. En este caso, también utilizaremos una máquina Ubuntu 22.04 con 2 vCPUs y 2 GB de RAM, que puedes personalizar según tus necesidades.
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
### Instalación de elastic search
En todos los nodos de nuestro cluster, debemos realizar una instalación base de Elasticsearch. Esto incluye registrar los repositorios necesarios, instalar Elasticsearch y configurar el nombre del cluster.
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
}
```
Ahora en el primer nodo  debemos habilitar y comenzar el servicio de Elasticsearch, generar un token de enrolamiento para los demás nodos y para Kibana. También cambiaremos la contraseña y la exportaremos a un archivo de texto, el cual podrás descargar en tu ordenador. Recuerda que este proceso solo es necesario realizarlo en el primer nodo.
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
```
Si hemos configurado más de un nodo, se ejecutara lo siguiente, subir el token y reconfigurar el nodo enrolandolo para que pueda ingresar al modo cluster.
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
¡Excelente! Ya hemos configurado nuestra cluster con Elasticsearch. Ahora es momento de configurar nuestro Kibana, la herramienta de visualización de datos. Para esto, debemos configurar los repositorios necesarios, instalar Kibana y, una vez iniciado, enrolar el sistema con nuestra cluster de Elasticsearch.
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
Al finalizar la configuración, se mostrará la URL de Kibana para que puedas conectarte y empezar a utilizar la herramienta. Recuerda que también se te proporcionará la contraseña en el archivo password.txt de tu cluster para que puedas acceder sin problemas.
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
  value       = "Your elastic password is ${file('password.txt')} can found in password.txt file"
  description = "Password elastic is: "
}
````
### Conclusión
Felicitaciones se ha instalado elastic search + kibana en tu insfraestructura para que lo puedas disfrutar en tus proyectos. Te invitamos a que puedas continuar utilizandolo en tus propios proyectos.

Te recomendamos que veas los siguientes proyectos:

1. Como instalar apm server utiliando elastic-agent con fleet server
1. Que base de datos es más rápida bases de datos relacionales vs elastic search

---
Authors:
  - [Manuel Alba](https://github.com/elmalba)
---

## License

[![CC0](http://mirrors.creativecommons.org/presskit/buttons/88x31/svg/cc-zero.svg)](https://creativecommons.org/publicdomain/zero/1.0/)

Our projects are built with the mindset of open-source applications, using the MIT license.
