```terraform
provider "digitalocean" {
  token = "<DIGITALOCEAN_API_TOKEN>"
}

resource "digitalocean_droplet" "postgresql1" {
  image  = "ubuntu-20-04-x64"
  name   = "postgresql1"
  region = "nyc3"
  size   = "s-2vcpu-2gb"

  ssh_keys = [
    "<YOUR_SSH_KEY_FINGERPRINT>",
  ]

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.postgresql1.ipv4_address
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y postgresql postgresql-contrib",
      "sudo sed -i 's/#listen_addresses = 'localhost'/listen_addresses = '*'/g' /etc/postgresql/13/main/postgresql.conf",
      "echo 'host all all 0.0.0.0/0 md5' | sudo tee -a /etc/postgresql/13/main/pg_hba.conf",
      "sudo systemctl restart postgresql",
    ]
  }
}

resource "digitalocean_droplet" "postgresql2" {
  image  = "ubuntu-20-04-x64"
  name   = "postgresql2"
  region = "nyc3"
  size   = "s-2vcpu-2gb"

  ssh_keys = [
    "<YOUR_SSH_KEY_FINGERPRINT>",
  ]

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.postgresql2.ipv4_address
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y postgresql postgresql-contrib",
      "sudo sed -i 's/#listen_addresses = 'localhost'/listen_addresses = '*'/g' /etc/postgresql/13/main/postgresql.conf",
      "echo 'host all all 0.0.0.0/0 md5' | sudo tee -a /etc/postgresql/13/main/pg_hba.conf",
      "sudo systemctl restart postgresql",
    ]
  }
}

resource "digitalocean_droplet" "postgresql3" {
  image  = "ubuntu-20-04-x64"
  name   = "postgresql3"
  region = "nyc3"
  size   = "s-2vcpu-2gb"

  ssh_keys = [
    "<YOUR_SSH_KEY_FINGERPRINT>",
  ]

  connection {
    type        = "ssh"
    user        = "root"
    private_key = file("~/.ssh/id_rsa")
    host        = digitalocean_droplet.postgresql3.ipv4_address
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y postgresql postgresql-contrib",
      "sudo sed -i 's/#listen_addresses = 'localhost'/listen_addresses = '*'/g' /etc/postgresql/13/main/postgresql.conf",
      "echo 'host all all 0.0.0.0/0 md5' | sudo tee -a /etc/postgresql/13/main/pg_hba.conf",
      "sudo systemctl restart postgresql",
    ]
  }
}

output
```
