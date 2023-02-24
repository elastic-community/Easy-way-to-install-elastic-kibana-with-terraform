# Not READY

# Easy-to-way-to-install-elastic-kibana-with-terraform
Easy way to install elastic search and kibana for adicional look guides "how to use install apm server with fleet server"


## Spanish Versión

La idea de este tutorial es demostrar como crear con un cluster con la cantidad de nodos que tu quieras de elastic search, un kibana sobre una cloud como digital ocean, esto es aplicable tambien para otras nubes cambiando los providers, aún asi te invito y te recomiendo usar elastic cloud https://cloud.elastic.co/home 


### Ejecuccion con terraform

Para instalar elastic con kibana nuestras infraestructura es tan simple como clonar este repositorio y ejecutar los siguiente comandos:
```bash
terraform init
terraform apply
```

Para mayor detalles te invito a continuar leyendo esta guia.

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

