# Not READY

# Easy-to-way-to-install-with-terraform
Easy way to install elastic search, kibana, apm 


## Spanish Versión

La idea deste tutorial es demostrar como crear con un cluster 3 nodos elastic search, un kibana y un servidor de apm sobre una cloud como digital ocean, esto es aplicable tambien para otras nubes, aún asi recomendamos que revices elastic cloud https://cloud.elastic.co/home 

### Elastic Search

Lo primero que haremos sera crear es instalar la base de datos de elastic search para cual con el siguiente manifiesto pueden ejecutar 

```terraform 

provider "digitalocean" {
  token = "<DIGITALOCEAN_API_TOKEN>"
}

resource "digitalocean_droplet" "server" {
  count  = 3
  image  = "ubuntu-20-04-x64"
  name   = "server-${count.index}"
  region = "nyc1"
  size   = "s-2vcpu-2gb"
}

resource "digitalocean_loadbalancer" "elasticsearch_lb" {
  name = "elasticsearch-lb"

  forwarding_rule {
    entry_port     = 9200
    entry_protocol = "tcp"

    target_port     = 9200
    target_protocol = "tcp"
  }

  healthcheck {
    port     = 9200
    protocol = "tcp"
  }

  droplet_ids = digitalocean_droplet.server.*.id
}

resource "null_resource" "install_elasticsearch" {
  count = 3

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.server[count.index].ipv4_address
  }

  provisioner "remote-exec" {
    inline = [
      "wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.15.2-linux-x86_64.tar.gz",
      "tar -xzf elasticsearch-7.15.2-linux-x86_64.tar.gz",
      "mv elasticsearch-7.15.2 /usr/local/elasticsearch",
      "echo 'network.host: 0.0.0.0' >> /usr/local/elasticsearch/config/elasticsearch.yml",
      "echo 'discovery.seed_hosts: [\"${join(",", digitalocean_droplet.server.*.ipv4_address)}\"]' >> /usr/local/elasticsearch/config/elasticsearch.yml",
      "echo 'cluster.initial_master_nodes: [\"node-1\"]' >> /usr/local/elasticsearch/config/elasticsearch.yml",
      "service elasticsearch restart"
    ]
  }
}

```

1. Ejecuta terraform init para inicializar el directorio de trabajo.
1. Ejecuta terraform apply para crear los servidores. Te pedirá que confirmes la creación, así que asegúrate de escribir "yes" cuando se te solicite.
1. Espera unos minutos mientras Terraform crea los servidores. Al finalizar, Terraform mostrará la información sobre los servidores creados.



### Kibana

```terraform
provider "digitalocean" {
  token = "<DIGITALOCEAN_API_TOKEN>"
}

resource "digitalocean_droplet" "kibana" {
  image  = "ubuntu-20-04-x64"
  name   = "kibana"
  region = "nyc3"
  size   = "s-2vcpu-2gb"

  ssh_keys = [
    "<YOUR_SSH_KEY_FINGERPRINT>",
  ]

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.kibana.ipv4_address
  }

  provisioner "remote-exec" {
    inline = [
      "curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -",
      "echo 'deb https://artifacts.elastic.co/packages/7.x/apt stable main' | sudo tee /etc/apt/sources.list.d/elastic-7.x.list",
      "sudo apt-get update",
      "sudo apt-get install -y kibana",
      "sudo sed -i 's/#server.host: \"localhost\"/server.host: \"0.0.0.0\"/' /etc/kibana/kibana.yml",
      "sudo systemctl enable kibana",
      "sudo systemctl start kibana",
    ]
  }
}
```


### APM Server

```terraform
provider "digitalocean" {
  token = "<DIGITALOCEAN_API_TOKEN>"
}

resource "digitalocean_droplet" "apm" {
  image  = "ubuntu-20-04-x64"
  name   = "apm"
  region = "nyc3"
  size   = "s-2vcpu-2gb"

  ssh_keys = [
    "<YOUR_SSH_KEY_FINGERPRINT>",
  ]

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.apm.ipv4_address
  }

  provisioner "remote-exec" {
    inline = [
      "curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -",
      "echo 'deb https://artifacts.elastic.co/packages/7.x/apt stable main' | sudo tee /etc/apt/sources.list.d/elastic-7.x.list",
      "sudo apt-get update",
      "sudo apt-get install -y apm-server",
      "sudo sed -i 's/#host: 0.0.0.0/host: 0.0.0.0/' /etc/apm-server/apm-server.yml",
      "sudo systemctl enable apm-server",
      "sudo systemctl start apm-server",
    ]
  }
}

Estamos listo para puedas ocupar elastic search en tu propia infraestructura ahora te invitamos a que puedas continuar utilizandolo en tus propios proyectos 

te recomendamos que veas los siguientes proyectos:

1.  [Como agregar observabilidad de elastic search a nuestro proyecto de next js](https://github.com/elastic-community/apm-elastic-next.js)
1.  [La mejor forma de generar observabilidad y monitoreo a nuestras aplicaciones en la nube ](https://github.com/elastic-community/ATP-elastic-APM)
1.  [Monitoreo de aplicaciones en una aplicación Golang con Elastic APM.](https://github.com/elastic-community/Golang-application-with-Elastic-APM)
