# Module Azure Virtual Machine

Module for creating and managing an Azure Virtual Machine.

## Usage

```
module "ssh-key" {
  source = "git::https://github.com/danielscholl-terraform/ssh-key?ref=v1.0.0"
}

data "template_cloudinit_config" "config" {
  gzip          = true
  base64_encode = true

  # Main cloud-config configuration file.
  part {
    content_type = "text/cloud-config"
    content      = "packages: ['httpie']"
  }
}

module "resource_group" {
  source = "git::https://github.com/danielscholl-terraform/module-resource-group?ref=v1.0.0"

  name     = "iac-terraform"
  location = "eastus2"

  resource_tags = {
    iac = "terraform"
  }
}

module "virtual_network" {
  source     = "git::https://github.com/danielscholl-terraform/module-virtual-network?ref=v1.0.0"
  depends_on = [module.resource_group]

  name                = "iac-terraform-vnet-${module.resource_group.random}"
  resource_group_name = module.resource_group.name

  dns_servers   = ["8.8.8.8"]
  address_space = ["192.168.1.0/24"]
  subnets = {
    vm-subnet = {
      cidrs = ["192.168.1.0/24"]

      allow_vnet_inbound      = true
      allow_vnet_outbound     = true
      allow_internet_outbound = true
    }
  }

  resource_tags = {
    iac = "terraform"
  }
}

module "virtual-machine" {
  source     = "git::https://github.com/danielscholl-terraform/module-virtual-machine?ref=v1.0.0"
  depends_on = [module.resource_group, module.virtual_network]

  resource_group_name  = module.resource_group.name
  name                 = "linux-vm"
  virtual_network_name = module.virtual_network.vnet.name
  subnet_name          = module.virtual_network.subnets["vm-subnet"].name
  ssh_key              = "${trimspace(module.ssh-key.public_ssh_key)} k8sadmin"

  os_type                  = "linux"
  linux_distribution_name  = "ubuntu2004"
  vm_size                  = "Standard_B2s"
  domain_name_label        = "iacterraformlinuxvm0"
  enable_public_ip_address = true

  custom_data = data.template_cloudinit_config.config.rendered

  resource_tags = {
    iac = "terraform"
  }
}

# Open SSH Port
resource "azurerm_network_security_rule" "allow_ssh" {
  name                        = "allow_ssh"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = "Internet"
  destination_address_prefix  = "VirtualNetwork"
  resource_group_name         = module.resource_group.name
  network_security_group_name = module.virtual_network.subnet_nsg_names["vm-subnet"]
}
```

<!--- BEGIN_TF_DOCS --->

<!--- END_TF_DOCS --->
