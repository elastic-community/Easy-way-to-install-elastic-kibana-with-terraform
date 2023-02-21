# Easy-to-way-to-install-with-terraform
Easy way to install elastic search, kibana, apm 


## Spanish Versión

La idea deste tutorial es demostrar como crear con un cluster 3 nodos elastic search, un kibana y un servidor de apm sobre una cloud como digital ocean, esto es aplicable tambien para otras nubes, aún asi recomendamos que revices elastic cloud https://cloud.elastic.co/home 

```terraform 

provider "digitalocean" {
  token = "<DIGITALOCEAN_API_TOKEN>"
}

resource "digitalocean_droplet" "server" {
  count  = 3
  image  = "ubuntu-18-04-x64"
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
      "/usr/local/elasticsearch/bin/elasticsearch &"
    ]
  }
}

```
