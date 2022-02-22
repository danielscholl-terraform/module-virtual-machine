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
## Providers

| Name | Version |
|------|---------|
| azurerm | >= 2.90.0 |
| random | n/a |
| tls | n/a |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:-----:|
| additional\_unattend\_content | The XML formatted content that is added to the unattend.xml file for the specified path and component. | `any` | n/a | yes |
| additional\_unattend\_content\_setting | The name of the setting to which the content applies. Possible values are `AutoLogon` and `FirstLogonCommands` | `any` | n/a | yes |
| admin\_password | The Password which should be used for the local-administrator on this Virtual Machine | `any` | n/a | yes |
| admin\_username | The username of the local administrator used for the Virtual Machine. | `string` | `"azureuser"` | no |
| custom\_data | Base64 encoded file of a bash script that gets run once by cloud-init upon VM creation | `any` | n/a | yes |
| custom\_image | Provide the custom image to this module if the default variants are not sufficient | <pre>map(object({<br>    publisher = string<br>    offer     = string<br>    sku       = string<br>    version   = string<br>  }))</pre> | n/a | yes |
| data\_disks | Managed Data Disks for azure viratual machine | <pre>list(object({<br>    name                 = string<br>    storage_account_type = string<br>    disk_size_gb         = number<br>  }))</pre> | `[]` | no |
| dedicated\_host\_id | The ID of a Dedicated Host where this machine should be run on. | `any` | n/a | yes |
| deploy\_log\_analytics\_agent | Install log analytics agent to windows or linux VM | `bool` | `false` | no |
| disable\_password\_authentication | Should Password Authentication be disabled on this Virtual Machine? Defaults to true. | `bool` | `true` | no |
| disk\_encryption\_set\_id | The ID of the Disk Encryption Set which should be used to Encrypt this OS Disk. The Disk Encryption Set must have the `Reader` Role Assignment scoped on the Key Vault - in addition to an Access Policy to the Key Vault | `any` | n/a | yes |
| disk\_size\_gb | The Size of the Internal OS Disk in GB, if you wish to vary from the size used in the image this Virtual Machine is sourced from. | `any` | n/a | yes |
| dns\_servers | List of dns servers to use for network interface | `list` | `[]` | no |
| domain\_name\_label | Label for the Domain Name. Will be used to make up the FQDN. If a domain name label is specified, an A DNS record is created for the public IP in the Microsoft Azure DNS system. | `any` | n/a | yes |
| enable\_accelerated\_networking | Should Accelerated Networking be enabled? Defaults to false. | `bool` | `false` | no |
| enable\_automatic\_updates | Specifies if Automatic Updates are Enabled for the Windows Virtual Machine. | `bool` | `true` | no |
| enable\_boot\_diagnostics | Should the boot diagnostics enabled? | `bool` | `false` | no |
| enable\_encryption\_at\_host | Should all of the disks (including the temp disk) attached to this Virtual Machine be encrypted by enabling Encryption at Host? | `bool` | `false` | no |
| enable\_ip\_forwarding | Should IP Forwarding be enabled? Defaults to false | `bool` | `false` | no |
| enable\_os\_disk\_write\_accelerator | Should Write Accelerator be Enabled for this OS Disk? This requires that the `storage_account_type` is set to `Premium_LRS` and that `caching` is set to `None`. | `bool` | `false` | no |
| enable\_proximity\_placement\_group | Manages a proximity placement group for virtual machines, virtual machine scale sets and availability sets. | `bool` | `false` | no |
| enable\_public\_ip\_address | Reference to a Public IP Address to associate with the NIC | `any` | n/a | yes |
| enable\_security\_group | Should a Network Security Group be created for this Virtual Machine? Defaults to true. | `bool` | `false` | no |
| enable\_ultra\_ssd\_data\_disk\_storage\_support | Should the capacity to enable Data Disks of the UltraSSD\_LRS storage account type be supported on this Virtual Machine | `bool` | `false` | no |
| enable\_vm\_availability\_set | Manages an Availability Set for Virtual Machines. | `bool` | `false` | no |
| existing\_network\_security\_group\_id | The resource id of existing network security group | `any` | n/a | yes |
| generate\_admin\_ssh\_key | Generates a secure private key and encodes it as PEM. | `bool` | `false` | no |
| instance\_count | The number of Virtual Machines required. | `number` | `1` | no |
| internal\_dns\_name\_label | The (relative) DNS Name used for internal communications between Virtual Machines in the same Virtual Network. | `any` | n/a | yes |
| key\_vault\_certificate\_secret\_url | The Secret URL of a Key Vault Certificate, which must be specified when `protocol` is set to `Https` | `any` | n/a | yes |
| license\_type | Specifies the type of on-premise license which should be used for this Virtual Machine. Possible values are None, Windows\_Client and Windows\_Server. | `string` | `"None"` | no |
| linux\_distribution\_list | Pre-defined Azure Linux VM images list | <pre>map(object({<br>    publisher = string<br>    offer     = string<br>    sku       = string<br>    version   = string<br>  }))</pre> | <pre>{<br>  "centos77": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "7.7",<br>    "version": "latest"<br>  },<br>  "centos78-gen2": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "7_8-gen2",<br>    "version": "latest"<br>  },<br>  "centos79-gen2": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "7_9-gen2",<br>    "version": "latest"<br>  },<br>  "centos81": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "8_1",<br>    "version": "latest"<br>  },<br>  "centos81-gen2": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "8_1-gen2",<br>    "version": "latest"<br>  },<br>  "centos82-gen2": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "8_2-gen2",<br>    "version": "latest"<br>  },<br>  "centos83-gen2": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "8_3-gen2",<br>    "version": "latest"<br>  },<br>  "centos84-gen2": {<br>    "offer": "CentOS",<br>    "publisher": "OpenLogic",<br>    "sku": "8_4-gen2",<br>    "version": "latest"<br>  },<br>  "coreos": {<br>    "offer": "CoreOS",<br>    "publisher": "CoreOS",<br>    "sku": "Stable",<br>    "version": "latest"<br>  },<br>  "mssql2019dev-rhel8": {<br>    "offer": "sql2019-rhel8",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "sqldev",<br>    "version": "latest"<br>  },<br>  "mssql2019dev-ubuntu1804": {<br>    "offer": "sql2019-ubuntu1804",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "sqldev",<br>    "version": "latest"<br>  },<br>  "mssql2019dev-ubuntu2004": {<br>    "offer": "sql2019-ubuntu2004",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "sqldev",<br>    "version": "latest"<br>  },<br>  "mssql2019ent-rhel8": {<br>    "offer": "sql2019-rhel8",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "enterprise",<br>    "version": "latest"<br>  },<br>  "mssql2019ent-ubuntu1804": {<br>    "offer": "sql2019-ubuntu1804",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "enterprise",<br>    "version": "latest"<br>  },<br>  "mssql2019ent-ubuntu2004": {<br>    "offer": "sql2019-ubuntu2004",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "enterprise",<br>    "version": "latest"<br>  },<br>  "mssql2019std-rhel8": {<br>    "offer": "sql2019-rhel8",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "standard",<br>    "version": "latest"<br>  },<br>  "mssql2019std-ubuntu1804": {<br>    "offer": "sql2019-ubuntu1804",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "standard",<br>    "version": "latest"<br>  },<br>  "mssql2019std-ubuntu2004": {<br>    "offer": "sql2019-ubuntu2004",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "standard",<br>    "version": "latest"<br>  },<br>  "rhel78": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "7.8",<br>    "version": "latest"<br>  },<br>  "rhel78-gen2": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "78-gen2",<br>    "version": "latest"<br>  },<br>  "rhel79": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "7.9",<br>    "version": "latest"<br>  },<br>  "rhel79-gen2": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "79-gen2",<br>    "version": "latest"<br>  },<br>  "rhel81": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "8.1",<br>    "version": "latest"<br>  },<br>  "rhel81-gen2": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "81gen2",<br>    "version": "latest"<br>  },<br>  "rhel82": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "8.2",<br>    "version": "latest"<br>  },<br>  "rhel82-gen2": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "82gen2",<br>    "version": "latest"<br>  },<br>  "rhel83": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "8.3",<br>    "version": "latest"<br>  },<br>  "rhel83-gen2": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "83gen2",<br>    "version": "latest"<br>  },<br>  "rhel84": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "8.4",<br>    "version": "latest"<br>  },<br>  "rhel84-byos": {<br>    "offer": "rhel-byos",<br>    "publisher": "RedHat",<br>    "sku": "rhel-lvm84",<br>    "version": "latest"<br>  },<br>  "rhel84-byos-gen2": {<br>    "offer": "rhel-byos",<br>    "publisher": "RedHat",<br>    "sku": "rhel-lvm84-gen2",<br>    "version": "latest"<br>  },<br>  "rhel84-gen2": {<br>    "offer": "RHEL",<br>    "publisher": "RedHat",<br>    "sku": "84gen2",<br>    "version": "latest"<br>  },<br>  "ubuntu1604": {<br>    "offer": "UbuntuServer",<br>    "publisher": "Canonical",<br>    "sku": "16.04-LTS",<br>    "version": "latest"<br>  },<br>  "ubuntu1804": {<br>    "offer": "UbuntuServer",<br>    "publisher": "Canonical",<br>    "sku": "18.04-LTS",<br>    "version": "latest"<br>  },<br>  "ubuntu1904": {<br>    "offer": "UbuntuServer",<br>    "publisher": "Canonical",<br>    "sku": "19.04",<br>    "version": "latest"<br>  },<br>  "ubuntu2004": {<br>    "offer": "0001-com-ubuntu-server-focal-daily",<br>    "publisher": "Canonical",<br>    "sku": "20_04-daily-lts",<br>    "version": "latest"<br>  },<br>  "ubuntu2004-gen2": {<br>    "offer": "0001-com-ubuntu-server-focal-daily",<br>    "publisher": "Canonical",<br>    "sku": "20_04-daily-lts-gen2",<br>    "version": "latest"<br>  }<br>}</pre> | no |
| linux\_distribution\_name | Variable to pick an OS flavour for Linux based VM. Possible values include: centos8, ubuntu1804 | `string` | `"ubuntu2004"` | no |
| log\_analytics\_customer\_id | The Workspace (or Customer) ID for the Log Analytics Workspace. | `any` | n/a | yes |
| log\_analytics\_workspace\_id | The name of log analytics workspace resource id | `any` | n/a | yes |
| log\_analytics\_workspace\_primary\_shared\_key | The Primary shared key for the Log Analytics Workspace | `any` | n/a | yes |
| managed\_identity\_ids | A list of User Managed Identity ID's which should be assigned to the Linux Virtual Machine. | `any` | n/a | yes |
| managed\_identity\_type | The type of Managed Identity which should be assigned to the Linux Virtual Machine. Possible values are `SystemAssigned`, `UserAssigned` and `SystemAssigned, UserAssigned` | `any` | n/a | yes |
| name | local name of the VM | `string` | n/a | yes |
| names | Names to be applied to resources (inclusive) | <pre>object({<br>    environment = string<br>    location    = string<br>    product     = string<br>  })</pre> | <pre>{<br>  "environment": "tf",<br>  "location": "eastus2",<br>  "product": "iac"<br>}</pre> | no |
| nsg\_diag\_logs | NSG Monitoring Category details for Azure Diagnostic setting | `list` | <pre>[<br>  "NetworkSecurityGroupEvent",<br>  "NetworkSecurityGroupRuleCounter"<br>]</pre> | no |
| nsg\_inbound\_rules | List of network rules to apply to network interface. | `list` | `[]` | no |
| os\_disk\_caching | The Type of Caching which should be used for the Internal OS Disk. Possible values are `None`, `ReadOnly` and `ReadWrite` | `string` | `"ReadWrite"` | no |
| os\_disk\_name | The name which should be used for the Internal OS Disk | `any` | n/a | yes |
| os\_disk\_storage\_account\_type | The Type of Storage Account which should back this the Internal OS Disk. Possible values include Standard\_LRS, StandardSSD\_LRS and Premium\_LRS. | `string` | `"StandardSSD_LRS"` | no |
| os\_type | Specify the flavor of the operating system image to deploy Virtual Machine. Valid values are `windows` and `linux` | `string` | `"linux"` | no |
| patch\_mode | Specifies the mode of in-guest patching to this Windows Virtual Machine. Possible values are `Manual`, `AutomaticByOS` and `AutomaticByPlatform` | `string` | `"AutomaticByOS"` | no |
| platform\_fault\_domain\_count | Specifies the number of fault domains that are used | `number` | `3` | no |
| platform\_update\_domain\_count | Specifies the number of update domains that are used | `number` | `5` | no |
| private\_ip\_address | The Static IP Address which should be used. This is valid only when `private_ip_address_allocation` is set to `Static` | `any` | n/a | yes |
| private\_ip\_address\_allocation\_type | The allocation method used for the Private IP Address. Possible values are Dynamic and Static. | `string` | `"Dynamic"` | no |
| public\_ip\_allocation\_method | Defines the allocation method for this IP address. Possible values are `Static` or `Dynamic` | `string` | `"Static"` | no |
| public\_ip\_availability\_zone | The availability zone to allocate the Public IP in. Possible values are `Zone-Redundant`, `1`,`2`, `3`, and `No-Zone` | `string` | `"Zone-Redundant"` | no |
| public\_ip\_sku | The SKU of the Public IP. Accepted values are `Basic` and `Standard` | `string` | `"Standard"` | no |
| public\_ip\_sku\_tier | The SKU Tier that should be used for the Public IP. Possible values are `Regional` and `Global` | `string` | `"Regional"` | no |
| random\_password\_length | The desired length of random password created by this module | `number` | `24` | no |
| resource\_group\_name | The name of the resource group in which the resources will be created | `string` | n/a | yes |
| resource\_tags | Map of tags to apply to taggable resources in this module. By default the taggable resources are tagged with the name defined above and this map is merged in | `map(string)` | `{}` | no |
| source\_image\_id | The ID of an Image which each Virtual Machine should be based on | `any` | n/a | yes |
| ssh\_key | specify the path to the existing SSH key to authenticate Linux virtual machine | `any` | n/a | yes |
| storage\_account\_name | The name of the hub storage account to store logs | `any` | n/a | yes |
| storage\_account\_uri | The Primary/Secondary Endpoint for the Azure Storage Account which should be used to store Boot Diagnostics, including Console Output and Screenshots from the Hypervisor. Passing a `null` value will utilize a Managed Storage Account to store Boot Diagnostics. | `any` | n/a | yes |
| subnet\_name | The name of the subnet to use in VM scale set | `string` | `""` | no |
| tags | A map of tags to add to all resources | `map(string)` | `{}` | no |
| virtual\_network\_name | The name of the virtual network | `string` | `""` | no |
| vm\_availability\_zone | The Zone in which this Virtual Machine should be created. Conflicts with availability set and shouldn't use both | `any` | n/a | yes |
| vm\_size | The Virtual Machine SKU for the Virtual Machine, Default is Standard\_A2\_V2 | `string` | `"Standard_D2_v2"` | no |
| vm\_time\_zone | Specifies the Time Zone which should be used by the Virtual Machine | `any` | n/a | yes |
| windows\_distribution\_list | Pre-defined Azure Windows VM images list | <pre>map(object({<br>    publisher = string<br>    offer     = string<br>    sku       = string<br>    version   = string<br>  }))</pre> | <pre>{<br>  "mssql2017dev": {<br>    "offer": "SQL2017-WS2019",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "sqldev",<br>    "version": "latest"<br>  },<br>  "mssql2017ent": {<br>    "offer": "SQL2017-WS2019",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "enterprise",<br>    "version": "latest"<br>  },<br>  "mssql2017exp": {<br>    "offer": "SQL2017-WS2019",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "express",<br>    "version": "latest"<br>  },<br>  "mssql2017std": {<br>    "offer": "SQL2017-WS2019",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "standard",<br>    "version": "latest"<br>  },<br>  "mssql2019dev": {<br>    "offer": "sql2019-ws2019",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "sqldev",<br>    "version": "latest"<br>  },<br>  "mssql2019ent": {<br>    "offer": "sql2019-ws2019",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "enterprise",<br>    "version": "latest"<br>  },<br>  "mssql2019ent-byol": {<br>    "offer": "sql2019-ws2019-byol",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "enterprise",<br>    "version": "latest"<br>  },<br>  "mssql2019std": {<br>    "offer": "sql2019-ws2019",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "standard",<br>    "version": "latest"<br>  },<br>  "mssql2019std-byol": {<br>    "offer": "sql2019-ws2019-byol",<br>    "publisher": "MicrosoftSQLServer",<br>    "sku": "standard",<br>    "version": "latest"<br>  },<br>  "windows2012r2dc": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2012-R2-Datacenter",<br>    "version": "latest"<br>  },<br>  "windows2016dc": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2016-Datacenter",<br>    "version": "latest"<br>  },<br>  "windows2016dccore": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2016-Datacenter-Server-Core",<br>    "version": "latest"<br>  },<br>  "windows2019dc": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2019-Datacenter",<br>    "version": "latest"<br>  },<br>  "windows2019dc-containers": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2019-Datacenter-with-Containers",<br>    "version": "latest"<br>  },<br>  "windows2019dc-containers-g2": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2019-datacenter-with-containers-g2",<br>    "version": "latest"<br>  },<br>  "windows2019dc-gensecond": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2019-datacenter-gensecond",<br>    "version": "latest"<br>  },<br>  "windows2019dc-gs": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2019-datacenter-gs",<br>    "version": "latest"<br>  },<br>  "windows2019dccore": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2019-Datacenter-Core",<br>    "version": "latest"<br>  },<br>  "windows2019dccore-g2": {<br>    "offer": "WindowsServer",<br>    "publisher": "MicrosoftWindowsServer",<br>    "sku": "2019-datacenter-core-g2",<br>    "version": "latest"<br>  }<br>}</pre> | no |
| windows\_distribution\_name | Variable to pick an OS flavour for Windows based VM. Possible values include: winserver, wincore, winsql | `string` | `"windows2019dc"` | no |
| winrm\_protocol | Specifies the protocol of winrm listener. Possible values are `Http` or `Https` | `any` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| admin\_ssh\_key\_private | The generated private key data in PEM format |
| admin\_ssh\_key\_public | The generated public key data in PEM format |
| linux\_virtual\_machine\_ids | The resource id's of all Linux Virtual Machine. |
| linux\_vm\_password | Password for the Linux VM |
| linux\_vm\_private\_ips | Public IP's map for the all windows Virtual Machines |
| linux\_vm\_public\_ips | Public IP's map for the all windows Virtual Machines |
| network\_security\_group\_ids | List of Network security groups and ids |
| vm\_availability\_set\_id | The resource ID of Virtual Machine availability set |
| windows\_virtual\_machine\_ids | The resource id's of all Windows Virtual Machine. |
| windows\_vm\_password | Password for the windows VM |
| windows\_vm\_private\_ips | Public IP's map for the all windows Virtual Machines |
| windows\_vm\_public\_ips | Public IP's map for the all windows Virtual Machines |
<!--- END_TF_DOCS --->
