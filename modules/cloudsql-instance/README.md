# Cloud SQL instance with read replicas

This module manages the creation of Cloud SQL instances with potential read replicas in other regions. It can also create an initial set of users and databases via the `users` and `databases` parameters.

Note that this module assumes that some options are the same for both the primary instance and all the replicas (e.g. tier, disks, labels, flags, etc).

*Warning:* if you use the `users` field, you terraform state will contain each user's password in plain text.

## Simple example

This example shows how to setup a project, VPC and a standalone Cloud SQL instance.

```hcl
module "project" {
  source          = "./modules/project"
  billing_account = var.billing_account_id
  parent          = var.organization_id
  name            = "my-db-project"
  services = [
    "servicenetworking.googleapis.com"
  ]
}

module "vpc" {
  source     = "./modules/net-vpc"
  project_id = module.project.project_id
  name       = "my-network"
  psa_config = {cloudsql-ew1-0 = {ranges = ["10.60.0.0/16"], routes = null}}
}

module "db" {
  source           = "./modules/cloudsql-instance"
  project_id       = module.project.project_id
  network          = module.vpc.self_link
  name             = "db"
  region           = "europe-west1"
  database_version = "POSTGRES_13"
  tier             = "db-g1-small"
}
# tftest modules=3 resources=6
```

## Cross-regional read replica

```hcl
module "db" {
  source           = "./modules/cloudsql-instance"
  project_id       = var.project_id
  network          = var.vpc.self_link
  name             = "db"
  region           = "europe-west1"
  database_version = "POSTGRES_13"
  tier             = "db-g1-small"

  replicas = {
    replica1 = "europe-west3"
    replica2 = "us-central1"
  }
}
# tftest modules=1 resources=3
```

## Custom flags, databases and users

```hcl
module "db" {
  source           = "./modules/cloudsql-instance"
  project_id       = var.project_id
  network          = var.vpc.self_link
  name             = "db"
  region           = "europe-west1"
  database_version = "MYSQL_8_0"
  tier             = "db-g1-small"

  flags = {
    disconnect_on_expired_password = "on"
  }

  databases = [
    "people",
    "departments"
  ]

  users = {
    # generatea password for user1
    user1 = null
    # assign a password to user2
    user2  = "mypassword"
  }
}
# tftest modules=1 resources=6
```
<!-- BEGIN TFDOC -->

## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [database_version](variables.tf#L50) | Database type and version to create. | <code>string</code> | ✓ |  |
| [name](variables.tf#L91) | Name of primary replica. | <code>string</code> | ✓ |  |
| [network](variables.tf#L96) | VPC self link where the instances will be deployed. Private Service Networking must be enabled and configured in this VPC. | <code>string</code> | ✓ |  |
| [project_id](variables.tf#L107) | The ID of the project where this instances will be created. | <code>string</code> | ✓ |  |
| [region](variables.tf#L112) | Region of the primary replica. | <code>string</code> | ✓ |  |
| [tier](variables.tf#L123) | The machine type to use for the instances. | <code>string</code> | ✓ |  |
| [authorized_networks](variables.tf#L17) | Map of NAME=>CIDR_RANGE to allow to connect to the database(s). | <code>map&#40;string&#41;</code> |  | <code>null</code> |
| [availability_type](variables.tf#L23) | Availability type for the primary replica. Either `ZONAL` or `REGIONAL`. | <code>string</code> |  | <code>&#34;ZONAL&#34;</code> |
| [backup_configuration](variables.tf#L29) | Backup settings for primary instance. Will be automatically enabled if using MySQL with one or more replicas. | <code title="object&#40;&#123;&#10;  enabled            &#61; bool&#10;  binary_log_enabled &#61; bool&#10;  start_time         &#61; string&#10;  location           &#61; string&#10;  log_retention_days &#61; number&#10;  retention_count    &#61; number&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  enabled            &#61; false&#10;  binary_log_enabled &#61; false&#10;  start_time         &#61; &#34;23:00&#34;&#10;  location           &#61; &#34;EU&#34;&#10;  log_retention_days &#61; 7&#10;  retention_count    &#61; 7&#10;&#125;">&#123;&#8230;&#125;</code> |
| [databases](variables.tf#L55) | Databases to create once the primary instance is created. | <code>list&#40;string&#41;</code> |  | <code>null</code> |
| [deletion_protection](variables.tf#L61) | Allow terraform to delete instances. | <code>bool</code> |  | <code>false</code> |
| [disk_size](variables.tf#L67) | Disk size in GB. Set to null to enable autoresize. | <code>number</code> |  | <code>null</code> |
| [disk_type](variables.tf#L73) | The type of data disk: `PD_SSD` or `PD_HDD`. | <code>string</code> |  | <code>&#34;PD_SSD&#34;</code> |
| [flags](variables.tf#L79) | Map FLAG_NAME=>VALUE for database-specific tuning. | <code>map&#40;string&#41;</code> |  | <code>null</code> |
| [labels](variables.tf#L85) | Labels to be attached to all instances. | <code>map&#40;string&#41;</code> |  | <code>null</code> |
| [prefix](variables.tf#L101) | Prefix used to generate instance names. | <code>string</code> |  | <code>null</code> |
| [replicas](variables.tf#L117) | Map of NAME=>REGION for additional read replicas. Set to null to disable replica creation. | <code>map&#40;any&#41;</code> |  | <code>null</code> |
| [users](variables.tf#L128) | Map of users to create in the primary instance (and replicated to other replicas) in the format USER=>PASSWORD. For MySQL, anything afterr the first `@` (if persent) will be used as the user's host. Set PASSWORD to null if you want to get an autogenerated password. | <code>map&#40;string&#41;</code> |  | <code>null</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [connection_name](outputs.tf#L24) | Connection name of the primary instance. |  |
| [connection_names](outputs.tf#L29) | Connection names of all instances. |  |
| [id](outputs.tf#L37) | ID of the primary instance. |  |
| [ids](outputs.tf#L42) | IDs of all instances. |  |
| [instances](outputs.tf#L50) | Cloud SQL instance resources. | ✓ |
| [ip](outputs.tf#L56) | IP address of the primary instance. |  |
| [ips](outputs.tf#L61) | IP addresses of all instances. |  |
| [self_link](outputs.tf#L69) | Self link of the primary instance. |  |
| [self_links](outputs.tf#L74) | Self links of all instances. |  |
| [user_passwords](outputs.tf#L82) | Map of containing the password of all users created through terraform. | ✓ |

<!-- END TFDOC -->
