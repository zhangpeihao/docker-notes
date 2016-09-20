# Docker Stacks与分布式应用群组包

[原文：《Docker Stacks and Distributed Application Bundles》](https://github.com/docker/docker/blob/master/experimental/docker-stacks-and-bundles.md)

## 概要

在Docker 1.12和Docker Compse 1.8中介绍了作为实验功能的Docker Stacks和分布式应用群组包, 随着Swarm模式概念和在Engine API中的Nodes与Services。

一个Dockerfile可以被构建成一个镜像，而容器则能通过镜像被创建出来。同样的，一个docker-compose.yml能被构建成一个 **分布式应用群组包** ，而 **Stacks** 则能通过这个群组包被创建出来。在这个意义上，群组包是一个多服务可分布式的镜像格式。

到Docker 1.12和Compose 1.8为止， 这个功能还是实验功能。Docker Engine和Docker Registry都还不支持分布式群组包。

## 生成一个群组包

生成一个群组包的最简单办法是使用`docker-compose`从一个现有的`docker-compose.yml`来创建。当然，这只是 *一个* 实现方法，类似的，`docker build`不是生成Docker镜像的唯一方法。

从`docker-compose`创建：

```bash
$ docker-compose bundle
WARNING: Unsupported key 'network_mode' in services.nsqd - ignoring
WARNING: Unsupported key 'links' in services.nsqd - ignoring
WARNING: Unsupported key 'volumes' in services.nsqd - ignoring
[...]
Wrote bundle to vossibility-stack.dab
```

> 译者注：Linux下docker-compose安装脚本：`curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose`

## 从一个群组包创建一个Stack

一个stack是用`docker deploy`命令来创建的：

```bash
# docker deploy --help

Usage:  docker deploy [OPTIONS] STACK

Create and update a stack from a Distributed Application Bundle (DAB)

Options:
      --file   string        Path to a Distributed Application Bundle file (Default: STACK.dab)
      --help                 Print usage
      --with-registry-auth   Send registry authentication details to Swarm agents
```

让我们发布之前创建的stack：

```bash
# docker deploy vossibility-stack
Loading bundle from vossibility-stack.dab
Creating service vossibility-stack_elasticsearch
Creating service vossibility-stack_kibana
Creating service vossibility-stack_logstash
Creating service vossibility-stack_lookupd
Creating service vossibility-stack_nsqd
Creating service vossibility-stack_vossibility-collector
```

我们可以确认一下服务是不是被正确地创建了：

```bash
# docker service ls
ID            NAME                                     REPLICAS  IMAGE
COMMAND
29bv0vnlm903  vossibility-stack_lookupd                1 nsqio/nsq@sha256:eeba05599f31eba418e96e71e0984c3dc96963ceb66924dd37a47bf7ce18a662 /nsqlookupd
4awt47624qwh  vossibility-stack_nsqd                   1 nsqio/nsq@sha256:eeba05599f31eba418e96e71e0984c3dc96963ceb66924dd37a47bf7ce18a662 /nsqd --data-path=/data --lookupd-tcp-address=lookupd:4160
4tjx9biia6fs  vossibility-stack_elasticsearch          1 elasticsearch@sha256:12ac7c6af55d001f71800b83ba91a04f716e58d82e748fa6e5a7359eed2301aa
7563uuzr9eys  vossibility-stack_kibana                 1 kibana@sha256:6995a2d25709a62694a937b8a529ff36da92ebee74bafd7bf00e6caf6db2eb03
9gc5m4met4he  vossibility-stack_logstash               1 logstash@sha256:2dc8bddd1bb4a5a34e8ebaf73749f6413c101b2edef6617f2f7713926d2141fe logstash -f /etc/logstash/conf.d/logstash.conf
axqh55ipl40h  vossibility-stack_vossibility-collector  1 icecrime/vossibility-collector@sha256:f03f2977203ba6253988c18d04061c5ec7aab46bca9dfd89a9a1fa4500989fba --config /config/config.toml --debug
```

## 管理Stacks

管理Stacks用`docker stack`管理：

```bash
# docker stack --help

Usage:  docker stack COMMAND

Manage Docker stacks

Options:
      --help   Print usage

Commands:
  config      Print the stack configuration
  deploy      Create and update a stack
  rm          Remove the stack
  services    List the services in the stack
  tasks       List the tasks in the stack

Run 'docker stack COMMAND --help' for more information on a command.
```

## 群组包文件格式

分布式应用群组包用Json格式来描述。当群组包作为文件形式保存，文件扩展名是`.bad`（Docker 1.12RC2工具还在使用`.dsb`扩展名，将会在下一个Release客户端中更新）。

一个群组包有两个顶级字段：`version`和`services`。在Docker 1.12工具中`version`是`0.1`。

在群组包中`services`是组成应用的服务。他们对应1.12版docker Engine API中介绍的新对象`Service`。

一个服务有以下一些字段：

```xml
<dl>
    <dt>
        Image (required) <code>string</code>
    </dt>
    <dd>
        The image that the service will run. Docker images should be referenced
        with full content hash to fully specify the deployment artifact for the
        service. Example:
        <code>postgres@sha256:f76245b04ddbcebab5bb6c28e76947f49222c99fec4aadb0bb
        1c24821a 9e83ef</code>
    </dd>
    <dt>
        Command <code>[]string</code>
    </dt>
    <dd>
        Command to run in service containers.
    </dd>
    <dt>
        Args <code>[]string</code>
    </dt>
    <dd>
        Arguments passed to the service containers.
    </dd>
    <dt>
        Env <code>[]string</code>
    </dt>
    <dd>
        Environment variables.
    </dd>
    <dt>
        Labels <code>map[string]string</code>
    </dt>
    <dd>
        Labels used for setting meta data on services.
    </dd>
    <dt>
        Ports <code>[]Port</code>
    </dt>
    <dd>
        Service ports (composed of <code>Port</code> (<code>int</code>) and
        <code>Protocol</code> (<code>string</code>). A service description can
        only specify the container port to be exposed. These ports can be
        mapped on runtime hosts at the operator's discretion.
    </dd>

    <dt>
        WorkingDir <code>string</code>
    </dt>
    <dd>
        Working directory inside the service containers.
    </dd>

    <dt>
        User <code>string</code>
    </dt>
    <dd>
        Username or UID (format: <code>&lt;name|uid&gt;[:&lt;group|gid&gt;]</code>).
    </dd>

    <dt>
        Networks <code>[]string</code>
    </dt>
    <dd>
        Networks that the service containers should be connected to. An entity
        deploying a bundle should create networks as needed.
    </dd>
</dl>
```

下面是一个有两个服务的bundlefile:

```json
{
	"Version": "0.1",
	"Services": {
		"redis": {
			"Image": "redis@sha256:4b24131101fa0117bcaa18ac37055fffd9176aa1a240392bb8ea85e0be50f2ce",
			"Networks": ["default"]
		},
		"web": {
			"Image": "dockercloud/hello-world@sha256:fe79a2cfbd17eefc344fb8419420808df95a1e22d93b7f621a7399fd1e9dca1d",
			"Networks": ["default"],
			"User": "web"
		}
	}
}
```

