---
layout: "rediscloud"
page_title: "Redis Cloud: rediscloud_subscription"
description: |-
  Subscription resource in the Terraform provider Redis Cloud.
---

# Resource: rediscloud_subscription

Creates a Subscription within your Redis Enterprise Cloud Account.
This resource is responsible for creating and managing subscriptions.

~> **Note:** The creation_plan block allows the API server to create a well-optimised infrastructure for your databases in the cluster.
The attributes inside the block are used by the provider to create initial 
databases. Those databases will be deleted after provisioning a new 
subscription, then the databases defined as separate resources will be attached to 
the subscription. The creation_plan block can ONLY be used for provisioning new 
subscriptions, the block will be ignored if you make any further changes or try importing the resource (e.g. `terraform import` ...).  

## Example Usage

```hcl
data "rediscloud_payment_method" "card" {
  card_type = "Visa"
}

resource "rediscloud_subscription" "subscription-resource" {

  name = "subscription-name"
  payment_method = "credit-card"
  payment_method_id = data.rediscloud_payment_method.card.id
  memory_storage = "ram"

  cloud_provider {
    provider = data.rediscloud_cloud_account.account.provider_type
    region {
      region = "eu-west-1"
      multiple_availability_zones = true
      networking_deployment_cidr = "10.0.0.0/24"
      preferred_availability_zones = ["euw1-az1, euw1-az2, euw1-az3"]
    }
  }

  // This block needs to be defined for provisioning a new subscription.
  // This allows creating a well-optimised hardware specification for databases in the cluster
  creation_plan {
    average_item_size_in_bytes = 1
    memory_limit_in_gb = 2
    quantity = 1
    replication= false
    support_oss_cluster_api= false
    throughput_measurement_by = "operations-per-second"
    throughput_measurement_value = 10000
    modules = ["RediSearch", "RedisBloom"]
  }
}
```

## Argument Reference

The following arguments are supported:

* `name` - (Required) A meaningful name to identify the subscription
* `payment_method` (Optional) The payment method for the requested subscription, (either `credit-card` or `marketplace`). If `credit-card` is specified, `payment_method_id` must be defined. Default: 'credit-card'
* `payment_method_id` - (Optional) A valid payment method pre-defined in the current account. This value is __Optional__ for AWS/GCP Marketplace accounts, but __Required__ for all other account types
* `memory_storage` - (Optional) Memory storage preference: either ‘ram’ or a combination of ‘ram-and-flash’. Default: ‘ram’
* `allowlist` - (Optional) An allowlist object, documented below 
* `cloud_provider` - (Required) A cloud provider object, documented below 
* `creation_plan` - (Required) A creation plan object, documented below

The `allowlist` block supports:

* `security_group_ids` - (Required) Set of security groups that are allowed to access the databases associated with this subscription
* `cidrs` - (Optional) Set of CIDR ranges that are allowed to access the databases associated with this subscription

~> **Note:** `allowlist` is only available when you run on your own cloud account, and not one that Redis provided (i.e `cloud_account_id` != 1)

The `cloud_provider` block supports:

* `provider` - (Optional) The cloud provider to use with the subscription, (either `AWS` or `GCP`). Default: ‘AWS’
* `cloud_account_id` - (Optional) Cloud account identifier. Default: Redis Labs internal cloud account
(using Cloud Account ID = 1 implies using Redis Labs internal cloud account). Note that a GCP subscription can be created
only with Redis Labs internal cloud account
* `region` - (Required) A region object, documented below

The `creation_plan` block supports:

* `memory_limit_in_gb` - (Required) Maximum memory usage that will be used for your largest planned database.
* `modules` - (Required) a list of modules that will be used by the databases in this subscription. Not currently compatible with ‘ram-and-flash’ memory storage.
Example: `modules = ["RedisJSON", RedisBloom"]`
* `support_oss_cluster_api` - (Optional) Support Redis open-source (OSS) Cluster API. Default: ‘false’
* `replication` - (Required) Databases replication. Set to `true` if any of your databases will use replication
* `quantity` - (Required) The planned number of databases in the subscription
* `throughput_measurement_by` - (Required) Throughput measurement method that will be used by your databases, (either ‘number-of-shards’ or ‘operations-per-second’)
* `throughput_measurement_value` - (Required) Throughput value that will be used by your databases (as applies to selected measurement method). The value needs to be the maximum throughput measurement value defined in one of your databases
* `average_item_size_in_bytes` - (Optional) Relevant only to ram-and-flash clusters
Estimated average size (measured in bytes) of the items stored in the database. The value needs to 
be the maximum average item size defined in one of your databases.  Default: 1000

~>**Note:** If the number of modules exceeds the `quantity` then additional creation-plan databases will be created with the modules defined in the `modules` block.

~> **Note:** If changes are made to attributes in the subscription which require the subscription to be recreated (such as `memory_storage`, `cloud_provider` or `payment_method`), the creation_plan will need to be defined in order to change these attributes. This is because the creation_plan is always required when a subscription is created.

The cloud_provider `region` block supports:

* `region` - (Required) Deployment region as defined by cloud provider
* `multiple_availability_zones` - (Optional) Support deployment on multiple availability zones within the selected region. Default: ‘false’
* `networking_deployment_cidr` - (Required) Deployment CIDR mask. The total number of bits must be 24 (x.x.x.x/24)
* `networking_vpc_id` - (Optional) Either an existing VPC Id (already exists in the specific region) or create a new VPC
(if no VPC is specified). VPC Identifier must be in a valid format (for example: ‘vpc-0125be68a4986384ad’) and existing
within the hosting account.
* `preferred_availability_zones` - (Required) Availability zones deployment preferences (for the selected provider & region). If multiple_availability_zones is set to 'true', you must select three availability zones from the list.

~> **Note:** The preferred_availability_zones parameter is required for Terraform, but is optional within the Redis Enterprise Cloud UI. 
This difference in behaviour is to guarantee that a plan after an apply does not generate differences. In AWS Redis internal cloud account, please set the zone IDs (for example: `["use-az2", "use-az3", "use-az5"]`).

### Timeouts

The `timeouts` block allows you to specify [timeouts](https://www.terraform.io/docs/configuration/resources.html#timeouts) for certain actions:

* `create` - (Defaults to 30 mins) Used when creating the subscription
* `update` - (Defaults to 30 mins) Used when updating the subscription
* `delete` - (Defaults to 10 mins) Used when destroying the subscription

## Attribute reference

The `region` block has these attributes:

* `networks` - List of generated network configuration

The `networks` block has these attributes:

* `networking_subnet_id` - The subnet that the subscription deploys into
* `networking_deployment_cidr` - Deployment CIDR mask for the generated
* `networking_vpc_id` - VPC id for the generated network

## Import

`rediscloud_subscription` can be imported using the ID of the subscription, e.g.

```
$ terraform import rediscloud_subscription.subscription-resource 12345678
```

~> **Note:** the creation_plan block will be ignored during imports.
