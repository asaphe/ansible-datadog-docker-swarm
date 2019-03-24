# Ansible Role Datadog-dockerized-agent

This role was a quick and dirty fix to a **specific** issue and **environment**. **Do not use it "as-is"**
Further more, Datadog has put little effort into the docker-swarm agent and seem to have abandon it.

- OS: Ubuntu 16.04+

build a container containing the Datadog agent and configuration files.
the image is based on Datadog's official Docker container and then pushed to a Docker Hub repository.
The role will also deploy that container using a template to create a Docker Stack file.

- Docker stack is globally replicated (One container per node)
- Docker stack is a template
- Dynamic checks use templates

> Auto-Discover at this time won't work over the network. meaning we can only auto-discover containers on the node we are running on and not the entire docker swarm.  

## Overview

- template the configuration
- build docker image
- push docker image
- deploy stack

>**Note:**The playbook that's running this role contains the basic logic what part of the role gets executed.
>Ansible Tags should be used when calling this role from Jenkins, even if the playbook isn't used.

## Updating a running Docker Stack

It takes the datadog-agent about 2 minutes to change from `starting` to `running`.
When updating the service (redeploying) it will update 4 containers at a time,
taking 2 minutes for all 4 to change to `running`.
A wait of an additional 10s to ensure there are no errors before moving to the next 4 containers.

## Usage Example

Ansible CLI example

```shell
ansible-playbook -i localhost -e env=prod -e build_docker_container=false -e credstash_aws_profile=prod-dynamic-inventory -e credstash_devops_table=devops
```

When using a Jenkins job it is highly recommended to use tags to target specific plays.
Available tags - Playbook level:
    tags: [datadog.dockerized.agent.remove.play]
    tags: [datadog.dockerized.agent.build.play]
    tags: [datadog.dockerized.agent.deploy.play]

### Defaults

The defaults file is populated using predefined values.

>the `defaults` part can be done much better (also the role :-) )