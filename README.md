# Supplemental Templates for AWS CloudFormation
Ambassador is an active user of the excellent `aws-cf-templates` and `cfn-modules` templates (link below) provided by 
widdix.  In a few cases, we've needed features that weren't supported by the core repository.  This repo shares these
templates back to the community (until and unless they're integrated into the core).

## Templates

This package includes the following templates

### Fargate: `service`, `cluster-alb-target`      

The core package offers two ways to expose a service, at least one of them is required:  through a cloudmap and through 
an ALB.  We had background services that needed neither and foreground services that needed both.  The `fargate/service` 
template in this package (also [PR #362](https://github.com/widdix/aws-cf-templates/pull/362)) accepts configurations 
for all or none of these service registries, automatically reconfiguring the service based on the presence or absence 
of parameters.

- The template is basically a drop-in replacement for `fargate/service-cloudmap` except that I renamed `ParentClientStack` to `ParentCloudMapIngressStack` and `Name` to `CloudMapName` to better link the parameter to the feature (could be reverted).
- To replicate the behavior of the existing `fargate/service-cluster-alb`, a user would combine `fargate/service` with the new `fargate/cluster-alb-target` shim:

	```
	  Service:
		Type: 'AWS::CloudFormation::Stack'
		Properties:
		  Parameters:
			...
			ParentALBTargetStack: !GetAtt 'ServiceALBTarget.Outputs.StackName'
			...
		  TemplateURL: 'aws-cf-templates/fargate/service.yaml'
	  ServiceALBTarget:
		Type: 'AWS::CloudFormation::Stack'
		Properties:
		  Parameters:
			...
		  TemplateURL: 'aws-cf-templates/fargate/cluster-alb-target.yaml'
	```

If needed, one could could create a shim for `service-dedicated-alb` as well.

### Cloudmap RDS Postgres

While most of our services use the CNAME for our RDS postgres server directly, we had two cases where we needed to 
expose the RDS server at a known address:

 - During the migration, so we could make and test smaller changes.
 - A legacy library that forces us to hard-code these kinds of parameters
 
This template creates a cloudmap entry for the database at the specified subdomain (e.g. db.\<local\>):

```
  CloudMapDB:
    Type: 'AWS::CloudFormation::Stack'
    Properties:
      Parameters:
        ParentCloudMapStack: !GetAtt 'CloudMap.Outputs.StackName'
        CloudMapHost: 'db'
        RdsPostgresStack: !GetAtt 'Database.Outputs.StackName'
      TemplateURL: './ambsw-cf-templates/state/cloudmap-rds-postgres.yaml'
```

### Step Function: Fargate Task with Retry

Our current best practice for migrations are step functions that run migrations in ECS Tasks.  This template 
demonstrates a state machine that runs the task and report success or failure.  Since we were using DockerHub images,
we often experienced a `CannotPullContainerError` that indicated some sort of pull error.  Since this was not a real 
failure, the state machine is configured to retry in response to this error.  **WARNING: Obviously this template cannot 
distinguish a case where the error is due to a legitimate problem like a container that does not exist.**     

### Fargate Task

To support out migration Step Function, we also need a definition for a Fargate Task.  This extracts the key Resources 
from Fargate Service (exposing the same Parameters) so they can be used in other instances like this. 

## Related projects

### aws-cf-templates
The core repository with Free Templates for AWS CloudFormation.

Learn more: https://github.com/widdix/aws-cf-templates

### cfn-modules
Rapid CloudFormation: Modular, production ready, open source also by widdix.

Learn more: https://github.com/cfn-modules/docs

### cfn-modules-shims
When Ambassador needed features that weren't available in `aws-cf-templates`, we looked to the related `cfn-moduels` 
library.  Since the interfaces for objects are dissimilar, we developed a variety of shims to make it easier to share
resources across the two libraries. 

Learn more: https://github.com/ambsw/cfn-modules-shims/

## License
All templates are published under Apache License Version 2.0.

## About
Ambassador Software Works develops and delivers EMR-integrated healthcare applications for hospitals and vendors.  The 
company's marquee APEXamenity product uses predictive analytics to identify patients at a high risk for poor experience.
Hospital staff (usually Nurse Leaders) leverage these insights to provide targeted mitigation efforts. 
