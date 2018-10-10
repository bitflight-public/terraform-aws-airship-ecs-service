# AWS ECS Service Terraform Module [![Build Status](https://travis-ci.org/blinkist/terraform-aws-airship-ecs-service.svg?branch=master)](https://travis-ci.org/blinkist/terraform-aws-airship-ecs-service)

![](https://raw.githubusercontent.com/blinkist/airship-tf-ecs-service/master/_readme_resources/airship.png)

## Introduction

ECS is AWS's original offering for Docker Orchestration. Although less feature rich than Kubernetes (EKS), it has proved to be an extremely stable platform for hosting stateless Docker services. This Terraform module is meant to be a one-size-fits-all ECS Service module. A module which makes it easy for any developer to create an ECS Service, have it attached to a load balancer, automatically grant it the necessary IAM permissions, and add extra scaling properties. By design it's not meant to update the ECS Services through Terraform once they have been created; rather, this is better handled by other open source projects like https://github.com/silinternational/ecs-deploy 

## Important notes

The role name belonging to the ECS Service has the following layout {clustername}-{servicename}-task-role , the combined length of the role name can be 64 Chars. Make sure that the combined length of your ECS cluster and service name is only 53 Chars.

### Application Load Balancer (ALB) attachment

![](https://raw.githubusercontent.com/blinkist/airship-tf-ecs-service/master/_readme_resources/alb_public.png)

By using the rule-based forwarding features of ALB, this module uses one ALB for many different microservices. For each ECS Service connected to a Load Balancer, a _Listener Rule_ is made based on the host-header (domain-name) of the ECS Service. Traffic is forwarded to them by the module-created _TargetGroup_ of the ECS Service.

When the module has ALB properties defined it will be connected to an Application Load Balancer by creating:
1. a `lb_listener_rule` based on the name of the service.
* 1a. (optional) a `lb_listener_rule` based on the variable `custom_listen_hosts`
2.  a route53 record inside the Route 53 Zone pointing to the load balancer.

This works for both externally visible services and for internally visible services. In this example we have:


```
  Company domain: mycorp.com

  Terraform development external route53 domain:     dev.mycorp.com
  Terraform development internal route53 domain: dev-int.mycorp.com
  
  == Internet-Facing ALB  *.dev.mycorp.com == 

  api.dev.mycorp. => api ecs service
  web.dev.mycorp. => web ecs service
```


### "Service Discovery" ALB Based

![](https://raw.githubusercontent.com/blinkist/airship-tf-ecs-service/master/_readme_resources/alb_discovery.png)

Unlike Kubernetes-style service discovery based on DNS, which lacks connection draining, ALB discovery adds a service to a load balancer and takes care of draining connections the moment an update takes place. One ALB can have multiple microservices as a backend by creating Layer 4-7 rules for the HTTP Host Header. Based on the `Host:` header, traffic will be forwarded to an ECS Service.

```
  [ name ] . [ route53_zone domain ]
```

In case `dev-int.mycorp.com` is used as domain for the internal ALB, the route53 records are being created which can be used by other ECS Services to connect to.
```
  == Internal ALB  *.dev-int.mycorp.com == 
  books.dev-int.mycorp. => micro1 ecs service
  mail.dev-int.mycorp. => micro2 ecs service
  micro3.dev-int.mycorp. => micro3 ecs service
  micro4.dev-int.mycorp. => micro4 ecs service
```

### KMS and SSM Management

AWS Systems Manager (also known as SSM) is the preferred way to store application parameters securely instead of using environment variables. The ECS module provides a way to give access to certain paths inside the [SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html). The full path which is given access to is being interpolated as such: "arn:aws:ssm:region:123456:parameter/application/%s/*". Parameters encrypted with KMS
will be automatically decrypted by most of the AWS libraries as long as the ECS Service also has access to the KMS key.

https://github.com/remind101/ssm-env

https://medium.com/@tdi/ssm-parameter-store-for-keeping-secrets-in-a-structured-way-53a25d48166a

### S3 Access

The module also provide simple access to S3 by the variables s3_ro_paths, and s3_rw_paths. In case the list is populated with S3 bucket names and folders, e.g. ["bucketname1/path","bucketname1/path2","bucketname3"], the module will ensure the ECS Service will have access to these resources, in either read only or read-write fashion, depending on if s3_ro_paths or s3_rw_paths have been used. Again, if KMS is used for encrypting S3 storage, the module need to be provided with the that KMS Key id.

### Cloudwatch logging

The default logging driver configured for the ECS Service is AWS Logging.

### Extra permissions

The Role ARN of the ECS Service is exported, and can be used to add other permissions e.g. to allow a service to make a cloudfront invalidation.

## Features
* [x] Can be conditionally created
* [x] Creates all necessary IAM Roles for running an ECS Service
* [x] Integrated IAM Permissions for KMS
* [x] Integrated IAM Permissions for SSM
* [x] Integrated IAM Permissions for S3
* [x] Creation of an ECS service, with/without AWSVPC, with/without FARGATE
* [x] Creation of ECS Task definition for use with/without AWSVPC, with/without FARGATE 
* [x] Integrated Cloudwatch Logging
* [x] Integrated Service Scaling
* [x] Handling of Creating listener rules to one ALB
* [x] Exports role arn for adding permissions 
* [x] Deregistration delay parameter allows for fast deployments
* [x] The scheduling_strategy DAEMON/REPLICA can be set 
* [x] Adding of Volumes / Mountpoints in case Docker Volume Drivers are used.
* [ ] ECS Service discovery
* [ ] Path based ALB Rules 
* [ ] SSL SNI Adding for custom hostnames
* [ ] Integrated IAM Permissions for *

## Simple ECS Service on Fargate with ALB Attached, together with a simple non ALB attached worker

```hcl

module "demo_web" {
  source  = "blinkist/airship-ecs-service/aws"
  version = "0.7.1"

  # The role name belonging to the ECS Service has the following layout {clustername}-{servicename}-task-role
  # Make sure that the combined length of your ECS cluster name and service name is only 53 Chars, with more that 53 chars
  # the length will surpass the maximum length of 64.

  name   = "demo-web"

  ecs_cluster_id = "${local.cluster_id}"

  region = "${local.region}"

  fargate_enabled = true

  # scheduling_strategy = "REPLICA"

  # AWSVPC Block, with awsvpc_subnets defined the network_mode for the ECS task definition will be awsvpc, defaults to bridge 
  awsvpc_enabled = true
  awsvpc_subnets            = ["${module.vpc.private_subnets}"]
  awsvpc_security_group_ids = ["${module.demo_sg.this_security_group_id}"]

  # load_balancing_enabled sets if a load balancer will be attached to the ecs service / target group
  load_balancing_enabled = true
  load_balancing_properties {
    # The default route53 record type, can be CNAME, ALIAS or NONE, currently CNAME to be backwards compatible
    # route53_record_type = "CNAME"

    # Unique identifier for the weighted IN A Alias Record 
    # route53_a_record_identifier = "identifier"

    # The ARN of the ALB, when left-out the service, will not be attached to a load-balance
    lb_arn                = "${module.alb_shared_services_ext.load_balancer_id}"
    # https listener ARN
    lb_listener_arn_https = "${element(module.alb_shared_services_ext.https_listener_arns,0)}"

    # http listener ARN
    lb_listener_arn       = "${element(module.alb_shared_services_ext.http_tcp_listener_arns,0)}"

    # The VPC_ID the target_group is being created in
    lb_vpc_id             = "${module.vpc.vpc_id}"

    # The route53 zone for which we create a subdomain
    route53_zone_id       = "${aws_route53_zone.shared_ext_services_domain.zone_id}"

    # After which threshold in health check is the task marked as unhealthy, defaults to 3
    # unhealthy_threshold   = "3"

    # health_uri defines which health-check uri the target group needs to check on for health_check, defaults to /ping
    # health_uri = "/ping"

    # The amount time for Elastic Load Balancing to wait before changing the state of a deregistering target from draining to unused. The range is 0-3600 seconds. 
    # deregistration_delay = "300"
  }

  # custom_listen_hosts defines extra listener rules to route to the ALB Targetgroup
  custom_listen_hosts    = ["www.example.com"]

  container_cpu    = 256
  container_memory = 512
  container_port   = 80
  container_image  = "nginx:latest"

  # Initial ENV Variables for the ECS Task definition
  container_envvars  {
       TASK_TYPE = "web" 
  } 

  # capacity_properties defines the size in task for the ECS Service.
  # Without scaling enabled, desired_capacity is the only necessary property, defaults to 2
  # With scaling enabled, desired_min_capacity and desired_max_capacity define the lower and upper boundary in task size
  capacity_properties {
    #desired_capacity     = "2"
    #desired_min_capacity = "2"
    #desired_max_capacity = "2"
  }

  # https://docs.aws.amazon.com/autoscaling/application/userguide/what-is-application-auto-scaling.html
  scaling_properties = [
    {
      type               = "CPUUtilization"
      direction          = "up"
      evaluation_periods = "2"
      observation_period = "300"
      statistic          = "Average"
      threshold          = "89"
      cooldown           = "900"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = "1"
    },
    {
      type               = "CPUUtilization"
      direction          = "down"
      evaluation_periods = "4"
      observation_period = "300"
      statistic          = "Average"
      threshold          = "10"
      cooldown           = "300"
      adjustment_type    = "ChangeInCapacity"
      scaling_adjustment = "-1"
    },
  ]

  # The KMS Keys which can be used for kms:decrypt
  kms_keys  = ["${module.global-kms.aws_kms_key_arn}", "${module.demo-kms.aws_kms_key_arn}"]

  # The SSM paths which are allowed to do kms:GetParameter and ssm:GetParametersByPath for
  #
  # https://medium.com/@tdi/ssm-parameter-store-for-keeping-secrets-in-a-structured-way-53a25d48166a
  # "arn:aws:ssm:region:123456:parameter/application/%s/*"
  #TODO
  ssm_paths = ["${module.global-kms.name}", "${module.demo-kms.name}"]

  # s3_ro_paths define which paths on S3 can be accessed from the ecs service in read-only fashion. 
  s3_ro_paths = []

  # s3_ro_paths define which paths on S3 can be accessed from the ecs service in read-write fashion. 
  s3_rw_paths = []
}

module "demo_web" {
  source  = "blinkist/airship-ecs-service/aws"
  version = "0.6.1"

  name   = "demo-worker"

  region         = "eu-central-1"

  ecs_cluster_id = "${module.ecs.cluster_id}"

  fargate_enabled = true
  awsvpc_enabled = true

  # scheduling_strategy = "REPLICA"

  # AWSVPC Block, with awsvpc_subnets defined the network_mode for the ECS task definition will be awsvpc, defaults to bridge 
  awsvpc_subnets            = ["${module.vpc.private_subnets}"]
  awsvpc_security_group_ids = ["${module.demo_sg.this_security_group_id}"]

  container_cpu    = 256
  container_memory = 512
  container_port   = 80
  container_image  = "nginx:latest"

  # Initial ENV Variables for the ECS Task definition
  container_envvars  {
       TASK_TYPE = "worker" 
  } 

  capacity_properties {
    desired_capacity     = "1"
  }

  kms_keys  = ["${module.global-kms.aws_kms_key_arn}", "${module.demo-kms.aws_kms_key_arn}"]
  ssm_paths = ["${module.global-kms.name}", "${module.demo-kms.name}"]
}


```

## Simple ECS Service on EC2-ECS with ALB Attached and no autoscaling

```hcl

module "demo_web" {
  source  = "blinkist/airship-ecs-service/aws"
  version = "0.6.1"

  name   = "demo5-web"

  ecs_cluster_id = "${local.cluster_id}"

  region         = "eu-central-1"

  # scheduling_strategy = "REPLICA""""

  # use_alb needs to be set to true
  load_balancing_enabled = true
  load_balancing_properties {
    # The default route53 record type, currently CNAME to be backwards compatible
    # route53_record_type = "CNAME"
    # Unique identifier for the weighted IN A Alias Record 
    # route53_record_identifier = "identifier"
    lb_arn                = "${module.alb_shared_services_ext.load_balancer_id}"
    lb_listener_arn_https = "${element(module.alb_shared_services_ext.https_listener_arns,0)}"
    lb_listener_arn       = "${element(module.alb_shared_services_ext.http_tcp_listener_arns,0)}"
    lb_vpc_id             = "${module.vpc.vpc_id}"
    route53_zone_id       = "${aws_route53_zone.shared_ext_services_domain.zone_id}"
    unhealthy_threshold   = "3"
    health_uri = "/ping"
  }

  container_cpu    = 256
  container_memory = 512
  container_port   = 80
  container_image  = "nginx:latest"

  # Initial ENV Variables for the ECS Task definition
  container_envvars  {
       SSM_ENABLED = "true"
       TASK_TYPE = "web" 
  } 

  # capacity_properties defines the size in task for the ECS Service.
  # Without scaling enabled, desired_capacity is the only necessary property
  # With scaling enabled, desired_min_capacity and desired_max_capacity define the lower and upper boundary in task size
  capacity_properties {
    desired_capacity     = "2"
  }

  # The KMS Keys which can be used for kms:decrypt
  kms_keys  = ["${module.global-kms.aws_kms_key_arn}", "${module.demo-kms.aws_kms_key_arn}"]

  # The SSM paths which are allowed to do kms:GetParameter and ssm:GetParametersByPath for
  ssm_paths = ["${module.global-kms.name}", "${module.demo-kms.name}"]
}

```

## Outputs
ecs_taskrole_arn - The ARN of the IAM Role for this task, can be used to add attach other IAM Permissions

