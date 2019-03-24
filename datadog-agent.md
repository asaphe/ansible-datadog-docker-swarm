# Datadog-Agent-V6

The dockerized version of the agent differs in a few ways from the standard agent which is meant to be installed directly on the host.
[further reading](https://github.com/DataDog/datadog-agent/tree/master/Dockerfiles/agent)
  
> The image is debian based

## Useful Agent commands

| Command | Description |
|:--------|:-----------:|
|`agent status`|show the current status, which checks are used etc,.|
|`agent configcheck`|show the currently loaded configuration|
|`agent configcheck -v`|show the currently loaded configuration, include configurations the agent weren't able to load or parse|
|`agent check ${nameOfCheck}`|manually run a the specified check|

>
  Use the `agent` command with no arguments to view the most current options

>
  *Checks* inside the datadog-agent container are under `/etc/datadog-agent/conf.d/`  
  **MOST** of the check paths are `/etc/datadog-agent/conf.d/${CHECKNAME}.d/`  
  **BUT** some of the checks are actually `/etc/datadog-agent/conf.d/${CHECKNAME}_check.d/`  
  this is something to be aware of when templating the check files

## Docker Swarm-Mode

To run the Datadog-agent in swarm mode and get all the relevant docker metrics we need to run it a service and make sure it's globally replicated like the following example:  

```dockerfile
deploy:
  mode: global
  restart_policy:
    condition: on-failure
    delay: 5s
    max_attempts: 3
  update_config:
    parallelism: 4
    delay: 10s
    failure_action: rollback
    order: stop-first
```

Notice we've also defined a `restart_policy` to ensure we always have the Datadog-agent containers up and running and we've defined an `update_config` which tells Docker Swarm-Mode to replace 4 containers at a time when updating the service, once a container is `running` it will wait `10s` before moving on to the next and it will always 4 containers at a time.

### Auto-Discover

> **enabled by default for the dockerized agent**  
> Auto-Discover at this time won't work over the network. meaning we can only auto-discover containers on the node we are running on and not the entire docker swarm.  

In a traditional non-container environment, Datadog Agent configuration is—like the environment in which it runs—static. The Agent reads check configurations from disk when it starts, and as long as it’s running, it continuously runs every configured check. The configuration files are static, and any network-related options configured within them serve to identify specific instances of a monitored service.  
[further reading](https://docs.datadoghq.com/agent/autodiscovery/)  

By default autodiscover uses the `Image Name` or `Container Name`.  
If we would like our template to run on our `Nginx` servers we simple use `- nginx`.  
It doesn't matter if the image used is `something/nginx:latest` it will match the `nginx` part.  

Example of a template meant to be used with autodiscovery:

```raw
ad_identifier:
  - nginx

init_config:

instances:
  - nginx_status_url: http://%%host%%:6060/nginx_status
    tags:
      - identifier:nginx
      - check:nginx_status
```

Afterwards, the agent will be able to pick up the variety of networks that are available based on the template variable you use. So in the above example, the %%host%% variable should cycle through available hosts in the network.
Static configuration files aren’t suitable for checks that collect data from ever-changing network endpoints, so Autodiscover uses templates for check configuration.  
In each template, the Agent looks for two template variables—`%%host%%` and `%%port%%`—to appear in place of any normally-hardcoded network options.  
Because templates don’t identify specific instances of a monitored service—which `%%host%%`? which `%%port%%`?—Autodiscover needs one or more container identifiers for each template so it can determine which IP(s) and Port(s) to substitute into the templates. **For Docker, container identifiers are image names or container labels.**  
Finally, Autodiscover can load check templates from places other than disk. Other possible template sources include key-value stores like Consul, and, when running on Kubernetes, Pod annotations.

#### Auto-Discovery Supported Template Variables

> a full list can be found [here](https://docs.datadoghq.com/agent/autodiscovery/)

| Variable | Description |
|:--------:|:------------|
|%%host%%|resolves to the IP|
|%%host_<NETWORK ID>%%|resolves to the ip on a specific network (ID or Name, docs are not clear. both?)|
|%%hostname%%|retrieves the hostname value from the container configuration *should not be used in most cases*|
|%%port%%|resolves to the highest exposed port sorted numerically and in ascending order (eg. 8443 for a container that exposes ports 80, 443, and 8443)|
|%%port_n%%|resolves to the first port sorted numerically and in ascending order|
|%%env_MYENVVAR%%|use the contents of the $MYENVVAR environment variable as seen by the Agent|

The above variables can be used in configuration templates where we've indicated we're using auto-discovery

#### Alternate Container Identifier: Labels
You can identify containers by label rather than container name or image.
label any container `com.datadoghq.ad.check.id: <SOME_LABEL>`, and then put `<SOME_LABEL>` anywhere you’d normally put a container name or image.  

```dockerfile
labels:
  com.datadoghq.ad.check.id: my-nginx
```

The above, defined on my nginx container will allow me to refer to that container in the config like so:

```dockerfile
ad_identifiers:
  - my-nginx
  - nginx
```

### Labels
Docker labels are very useful for the team and can be used by the Datadog-agent to dynamically define checks and affect auto-discover without using configuration files.
> Docker labels can be added as tags - specify it using the config file or environment variables  

**In A Docker-Stack(Compose) file** (will be added to the service when it is deployed)

```dockerfile
labels:
  com.datadoghq.ad.check_names: '[<CHECK_NAME>]'
  com.datadoghq.ad.init_configs: '[<INIT_CONFIG>]'
  com.datadoghq.ad.instances: '[<INSTANCE_CONFIG>]'
  com.datadoghq.ad.logs: '[<LOGS_CONFIG>]'
```

**In A Dockerfile** (will be added to the container when we build the image)

```dockerfile
LABEL "com.datadoghq.ad.check_names"='[<CHECK_NAME>]'
LABEL "com.datadoghq.ad.init_configs"='[<INIT_CONFIG>]'
LABEL "com.datadoghq.ad.instances"='[<INSTANCE_CONFIG>]'
LABEL "com.datadoghq.ad.logs"='[<LOGS_CONFIG>]'
```

**Docker Example: Nginx Dockerfile**

```dockerfile
FROM nginx

EXPOSE 8080
COPY nginx.conf /etc/nginx/nginx.conf
LABEL "com.datadoghq.ad.check_names"='["nginx"]'
LABEL "com.datadoghq.ad.init_configs"='[{}]'
LABEL "com.datadoghq.ad.instances"='[{"nginx_status_url": "http://%%host%%/nginx_status:%%port%%"}]'
LABEL "com.datadoghq.ad.logs"='[{"source": "nginx", "service": "webapp"}]'
```

### Tags

*[further reading](https://docs.datadoghq.com/getting_started/tagging/)*  

After you have assigned tags at the host and integration level, you can start using them to filter and group in interesting ways. There are several places you can use tags:  
    - Events List  
    - Dashboards  
    - Infrastructure List  
    - Host Map  
    - Monitors  
Tagging is used throughout the Datadog product to make it easier to subset and query the machines and metrics that you have to monitor. Without the ability to assign and filter based on tags, finding the problems that exist in your environment and narrowing them down enough to discover the true causes would be extremely difficult.  

**Docker Tags:**  

automatically collect common tags from Docker, Kubernetes, ECS, Swarm, Mesos, Nomad and Rancher, and allow you to extract even more tags with the following options:

```shell
  DD_DOCKER_LABELS_AS_TAGS : extract docker container labels
  DD_DOCKER_ENV_AS_TAGS : extract docker container environment variables
  DD_KUBERNETES_POD_LABELS_AS_TAGS : extract pod labels
```

**Tagging guidelines:**

* *must* start with a letter
* can contain: alphanumerics[a-zA-Z0-9], underscores[_], hyphens[-], colons[:], periods[.], slashes[\/]
* other special characters get converted to underscores. Note: A tag cannot end with a colon (e.g., tag:)
* tags are converted to lowercase
* up to 200 characters long in unicode
* tags can have a `value` or `key:value`. key-value is the best practice (makes sense)

  ```shell
    - role:database:mysql is parsed as key:role , value:database:mysql
    - role_database:mysql is parsed as key:role_database , value:mysql
  ```

  Examples of commonly used metric tag keys are env, instance, name, and role.

* reserved tags:
  * device
  * host
  * source
* tags shouldn’t originate from unbounded sources, such as EPOCH timestamps or user IDs. These tags may impact platform performance and billing.

>DO NOT use spaces when setting up tags.

```shell
  tags:
    - mytag:myvalue
```

**Applying tags:**

Tags can be added to the following:

- Agent via `datadog.yaml`
- Checks/Integration/Templates (edit the yaml)
- Tags generated by other services such as AWS, Docker
- Manually adding tags using the Infrastructure List  
*there are a few more ways to tag or where tags can be defined, i.e endpoints*  

### Including or Excluding containers

**Configured via ENV Vars or `datadog.yaml`**  

By default everything is included. we can use the following to determine what we monitor.

```shell
DD_AC_INCLUDE: whitelist of containers to always include
DD_AC_EXCLUDE: blacklist of containers to exclude
```

The format for these option is space-separated strings. For example:

```shell
DD_AC_EXCLUDE = "image:.*"
DD_AC_INCLUDE = "image:cp-kafka image:k8szk"
```

### Env-vars

Datadog's agent can have multiple settings defined using enviroment variables defined in our docker-stack file.
example:

```shell
environment:
 - DD_API_KEY={{ datadog_apiKey }}
 - DD_APM_ENABLED=false
 - DD_LOGS_ENABLED=false
 - DD_DOCKER_LABELS_AS_TAGS='{"com.devops.service":"service_name", "dvps.app":"service_name", "dvps.server.version":"server_version", "dvps.stack":"stack_name"}'
 - DOCKER_DD_AGENT=yes
```

it's straight forward, further info can be found [here](https://docs.datadoghq.com/agent/basic_agent_usage/docker/)

### Mounts

When running as service we need to mount the host directories on each node to the container running on it.
this differs a bit from running a standalone container.  

```shell
volumes:
 - type: bind
   source: /var/run/docker.sock
   target: /var/run/docker.sock
   read_only: true
 - type: bind
   source: /proc/
   target: /host/proc/
   read_only: true
 - type: bind
   source: /sys/fs/cgroup/
   target: /host/sys/fs/cgroup
   read_only: true
```

### Networks

In order to run checks on other containers and effectively monitor the swarm we need to ensure our Datadog-Agent container has a 'leg' in each network.  
the agent needs to be able to reach other containers and services using DNS resolution

```shell
networks:
  - default
  - prod_new_default
  - prod_new_tasks_default
```

### Configuration files

The official instruction for the agent will tell you to put files in `/conf.d/` and they will be copied to `/etc/datadog-agent/conf.d` which in turn should include
everything under that path.

> The `conf.d` location didn't work for me as expected

we can potentially have the following structure:

```shell
- /etc/datadog-agent/conf.d/ntp.d/
  |-/prod/conf.yaml
  |-/dev/conf.yaml
```

That way we can have different configurations with different settings for each enviroment and use a single image for both.
The filename doesn't even have to be `conf.yaml`, meaning you can also have the following:

```shell
- /etc/datadog-agent/conf.d/ntp.d/
  |-prod.yaml
  |-dev.yaml
```

**Agent Directory:**

> this is where the binaries are and it's part of ${PATH}; the `agent` command can be called from anywhere in the system or when running docker exec without specifying the full path to the binary

```shell
/opt/datadog-agent/
```

**Agent Config:**

```shell
/etc/datadog-agent/datadog.yml
```

**Configure checks:**

```shell
- /etc/datadog-agent/conf.d/<check_name>/conf.yaml
- /etc/datadog-agent/conf.d/<check_name>/conf.yaml.default
```

Usually you will just override the `conf.yaml` file.
Some checks have a `.default` extension wich will override any other. either remove that file or override it.

```shell
/etc/datadog-agent/conf.d/cpu.d/conf.yaml.default
/etc/datadog-agent/conf.d/disk.d/conf.yaml.default
/etc/datadog-agent/conf.d/docker.d/conf.yaml.default
/etc/datadog-agent/conf.d/file_handle.d/conf.yaml.default
/etc/datadog-agent/conf.d/io.d/conf.yaml.default
/etc/datadog-agent/conf.d/load.d/conf.yaml.default
/etc/datadog-agent/conf.d/memory.d/conf.yaml.default
/etc/datadog-agent/conf.d/network.d/conf.yaml.default
/etc/datadog-agent/conf.d/ntp.d/conf.yaml.default
/etc/datadog-agent/conf.d/uptime.d/conf.yaml.default
```

### Configuring Templates

Templates are everything under `/etc/datadog-agent/conf.d/` and define how we use pre-defined checks. *Datadog* referes to them as `templates`. there's also `checks` which should reside under a different path and are scripts written by the customer (Py).

> Storing templates as local files is easy to understand and doesn’t require an external service or a specific orchestration platform. The downside is that you have to restart your Agent containers each time you change, add, or remove templates.
> The alternative is to use Docker labels on the containers we want to monitor. see [labels](#labels) for more.

#### Template Source Precedence

If you provide a template for the same check type via multiple template sources, the Agent looks for templates in the following order (using the first one it finds):  

- Kubernetes annotations/Docker labels
- Files

### Custom Checks

Custom [checks](https://docs.datadoghq.com/developers/agent_checks/#configuration) are python scripts written by the customer to query/monitor/get a result and pass it to datadog.
Check reside in `/etc/datadog-agent/checks.d/` and can have a similar structure to templates.

> custom checks cannot have the same name as a existing check or library within the Agent’s embedded environment.
