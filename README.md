
# easy-to-way-to-install-elastic-kibana-with-terraform
Easy way to install elastic search and kibana for adicional guides look "how to use install apm server with fleet server" this post is available in english and [spanish](https://github.com/elastic-community/Easy-to-way-to-install-elastic-kibana-with-terraform/README-ES.md)

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


### Ready
Congratulations, you have installed Elasticsearch + Kibana on your infrastructure so that you can enjoy it in your projects. We invite you to continue using it in your own projects.

We recommend that you check out the following projects:

---
Authors:
  - [Manuel Alba](https://github.com/elmalba)
  - [Gustavo Gonzalez](https://github.com/GGonzalezRojas)
---

## License

[![CC0](http://mirrors.creativecommons.org/presskit/buttons/88x31/svg/cc-zero.svg)](https://creativecommons.org/publicdomain/zero/1.0/)

Our projects are built with the mindset of open-source applications, using the MIT license.
