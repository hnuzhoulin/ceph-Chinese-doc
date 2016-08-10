================
 配置联合网关
================

.. versionadded:: 0.67 Dumpling

Ceph 从 0.67 Dumpling 起，支持将:term:`Ceph 对象网关`\ 添加到一个多区域或者\
  多可用区的联合架构中。

- **Region**: 区域是地理空间的\ *逻辑*\ 划分，它包含一个或多个可用区。一个\
  包含多个区域的集群必须指定一个主区域。

- **Zone**: 可用区是一个或多个 Ceph 对象网关实例的\ *逻辑*\ 分组。每个区域\
  有一个主可用区处理客户端请求。

.. important:: 你只应该向一个区域的主可用区写入对象，当然，可以从副本可用区\
  读取对象。当前，网关程序不会阻止你写入副本可用区，但是，\ **别那样干！**


背景
====

当你部署一个跨地理位置的 :term:`Ceph 对象存储`\ 服务时，（你应该）配置 Ceph \
对象网关区域及元数据同步代理以实现全局统一的命名空间，即使 Ceph 对象网关实例\
运行在不同的地理位置、甚至不同的 Ceph 集群上。当你想把一个区域内的一个或多个\
 Ceph 对象网关实例拆分到多个逻辑容器以维护一份或者多份额外的数据拷贝时，\
（你应该）配置 Ceph 对象网关可用区及数据同步代来维护主可用区数据的一份或多份\
数据副本。额外的数据副本对于故障转移、备份和灾难恢复很重要。

如果你有低延时的网络连接，就可以部署一个联合架构的单体集群（不建议这么做）。\
也可以在每个区域部署一套 Ceph 存储集群，并给每个可用区配置不同的存储池\
（典型部署）。你也可以在每个可用区都部署一个独立的 Ceph 存储集群，如果你的要求\
和资源保证了这种级别的冗余。


关于本指南
==========

在下面的章节，我们将演示如何用两个逻辑步骤配置一个联合集群：

- **配置主区域：** 这一节描述了如何配置一个包含多个可用区的区域 ，以及\
  如何在主区域内的主可用区和各次可用区之间同步数据。

- **配置次区域：** 这一节描述了如何重复前一节中配置一个包含多个可用区的主\
  区域过程，这样你就有了两个实现了区域内可用区间同步的区域。最后，你会学到\
  如何配置元数据同步代理，以此统一集群中各区域的命名空间。


配置主区域
=============

本节提供一个配置包含两个可用区的区域的典型步骤，此集群将包含两个网关守\
护进程——每个可用区一个。此区域将作为主区域 。


为主区域命名
----------------

配置集群前，规划良好的区域 、可用区和实例名字可帮你更好地管理集群。假设此 \
区域代表美国，那我们就引用她的标准缩写。

- United States: ``us``

假设两个可用区分别为美国的东部和西部，为保持连贯性，命名将采用 \
``{region name}-{zone name}`` 格式，但你可以用自己喜欢的命名规则。

- United States, East Region: ``us-east``
- United States, West Region: ``us-west``

最后，我们假设每个可用区都配置了至少一个 Ceph 对象网关实例，为保持连贯性，我们将\
按 ``{region name}-{zone name}-{instance}`` 格式命名，但你可以用自己喜欢的命\
名规则。

- United States Region, Master Zone, Instance 1: ``us-east-1``
- United States Region, Secondary Zone, Instance 1: ``us-west-1``


创建存储池
----------

你可以把整个区域配置为一个 Ceph 存储集群，也可以为各个可用区都配置为一个 \
Ceph 存储集群。

为保持连贯性，我们将 ``{region name}-{zone name}`` 作为存储池名的前缀，\
但你可以用自己喜欢的命名规则。例如：


- ``.us-east.rgw``
- ``.us-east.rgw.root``
- ``.us-east.rgw.control``
- ``.us-east.rgw.gc``
- ``.us-east.rgw.buckets``
- ``.us-east.rgw.buckets.index``
- ``.us-east.rgw.buckets.extra``
- ``.us-east.log``
- ``.us-east.intent-log``
- ``.us-east.usage``
- ``.us-east.users``
- ``.us-east.users.email``
- ``.us-east.users.swift``
- ``.us-east.users.uid``

|

- ``.us-west.rgw``
- ``.us-west.rgw.root``
- ``.us-west.rgw.control``
- ``.us-west.rgw.gc``
- ``.us-west.rgw.buckets``
- ``.us-west.rgw.buckets.index``
- ``.us-west.rgw.buckets.extra``
- ``.us-west.log``
- ``.us-west.intent-log``
- ``.us-west.usage``
- ``.us-west.users``
- ``.us-west.users.email``
- ``.us-west.users.swift``
- ``.us-west.users.uid``

关于网关的默认存储池请参考\ `配置参考——存储池`_\ 。关于创建存储池见\ \
`存储池`_\ 。用下列命令创建存储池： ::

	ceph osd pool create {poolname} {pg-num} {pgp-num} {replicated | erasure} [{erasure-code-profile}] {ruleset-name} {ruleset-number}


.. tip:: 创建大量存储池时，集群回到 ``active + clean`` 状态可能需要较长的时间。

.. topic:: CRUSH Maps

	把整个区域配置为一个 Ceph 存储集群时，请考虑为可用区使用独立的 CRUSH 规则，\
	这样就不会有重叠的故障域了。详情见 `CRUSH 图`_\ 。

	Ceph 允许多级 CRUSH 和多种 CRUSH 规则集，这样在配置你自己的网关时就\
	有很大的灵活性。像 ``rgw.buckets.index`` 这样的存储池就可以利用适当\
	大小的 SSD 存储池取得高性能；后端存储也可以从更经济的纠删码存储中获益，\
	还可能利用缓存层提升性能。

完成这一步后，执行下列命令以确认你已经创建了前述所需的存储池： ::

	rados lspools


创建密钥环
----------

各实例都必须有用户名和密钥才能与 Ceph 存储集群通信。在下面几步，我们用管理节\
点创建密钥环。然后为各实例创建客户端用户名及其密钥。接着把这些密钥加入 Ceph 存\
储集群。最后，把密钥环分发给各网关节点。

#. 创建密钥环。 ::

	sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.radosgw.keyring
	sudo chmod +r /etc/ceph/ceph.client.radosgw.keyring


#. 为各实例生成 Ceph 对象网关用户名及密钥。 ::

	sudo ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.us-east-1 --gen-key
	sudo ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.us-west-1 --gen-key


#. 给各密钥增加权限，允许到监视器的写权限、允许创建存储池，详情见\ \
   `配置参考——存储池`_\ 。 ::

	sudo ceph-authtool -n client.radosgw.us-east-1 --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
	sudo ceph-authtool -n client.radosgw.us-west-1 --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring


#. 创建密钥环及密钥后，要授权 Ceph 对象网关访问 Ceph 存储集群，还需把各密钥\
   导入存储集群。例如： ::

	sudo ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.us-east-1 -i /etc/ceph/ceph.client.radosgw.keyring
	sudo ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.us-west-1 -i /etc/ceph/ceph.client.radosgw.keyring


.. note:: 按照以上步骤配置二级次区域时，需把 ``us-`` 替换为 ``eu-`` 。\
   创建主区域和次区域**后**\ ，你一共会拥有四个用户。


安装 Apache 和 FastCGI
----------------------

每个运行 :term:`Ceph 对象网关`\ 守护进程实例的 :term:`Ceph 节点`\ 都必须安装 \
Apache 、 FastCGI 、 Ceph 对象网关守护进程（ ``radosgw`` ），以及 Ceph 对象网\
关同步代理（ ``radosgw-agent`` ）。详情见\ `安装 Ceph 对象网关`_\ 。


创建数据目录
------------

分别为各主机上的各个守护进程实例创建数据目录。 ::

	ssh {us-east-1}
	sudo mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.us-east-1

	ssh {us-west-1}
	sudo mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.us-west-1


.. note:: 按照以上步骤配置次区域时，需把 ``us-`` 替换为 ``eu-`` 。\
   创建主区域和次区域 **后**\ ，你一共会拥有四个数据目录。


创建网关配置
------------

在装了 Ceph 对象网关守护进程的主机上，为各实例创建 Ceph 对象网关配置文件，并\
放到 ``/etc/apache2/sites-available`` 目录下。典型的网关配置见下文：

.. literalinclude:: rgw.conf
   :language: ini

#. 把 ``/{path}/{socket-name}`` 条目分别替换为套接字的路径和名字，例如 \
   ``/var/run/ceph/client.radosgw.us-east-1.sock`` ，确保 ``ceph.conf`` 里也\
   使用相同的套接字路径和名字。

#. 用服务器的全域名替换 ``{fqdn}`` 条目。

#. 用服务器管理员的邮件地址替换 ``{email.address}`` 条目。

#. 如果你想用 S3 风格的子域，需添加 ``ServerAlias`` 配置（你当然想）。

#. 把这些配置另存到一文件，如 ``rgw-us-east.conf`` 。

为次区域重复这些步骤，如 ``rgw-us-west.conf`` 。

.. note:: 按照以上步骤配置次区域时，需把 ``us-`` 替换为 ``eu-`` 。\
   创建主区域和次区域 **后**\ ，你一共会有四个网关配置文件在相应的节点上。


最后，如果你想启用 SSL ，确保端口要设置为 SSL 端口（通常是 443 ），而且配置文\
件要包含这些： ::

	SSLEngine on
	SSLCertificateFile /etc/apache2/ssl/apache.crt
	SSLCertificateKeyFile /etc/apache2/ssl/apache.key
	SetEnv SERVER_PORT_SECURE 443


启用配置
--------

启用各实例的网关配置、并禁用默认站点。

#. 启用网关站点配置。 ::

	sudo a2ensite {rgw-conf-filename}

#. 禁用默认站点。 ::

	sudo a2dissite default

.. note:: 默认站点禁用失败会导致其他问题。


添加 FastCGI 脚本
-----------------

要启用 S3 兼容接口，各 Ceph 对象网关实例都需要一个 FastCGI 脚本。按如下过程创\
建此脚本。

#. 进入 ``/var/www`` 目录。 ::

	cd /var/www

#. 用编辑器打开名为 ``s3gw.fcgi`` 的文件。\ **注：**\ 配置文件里也要指定此文\
   件名。 ::

	sudo vim s3gw.fcgi

#. 添加 shell 脚本头，然后是 ``exec`` 、网关的二进制文件路径、 Ceph 配置文件路\
   径、还有用户名（ ``-n`` 所指，就是\ `创建密钥环`_\ 里第二步所创建的用户），把\
   下面的内容复制进编辑器。 ::

	#!/bin/sh
	exec /usr/bin/radosgw -c /etc/ceph/ceph.conf -n client.radosgw.{ID}

   例如： ::

	#!/bin/sh
	exec /usr/bin/radosgw -c /etc/ceph/ceph.conf -n client.radosgw.us-east-1

#. 保存文件。

#. 增加可执行权限。 ::

	sudo chmod +x s3gw.fcgi


在次区域上重复以上步骤。

.. note:: 按照以上步骤配置次区域时，需把 ``us-`` 替换为 ``eu-`` 。\
   创建主区域和次区域 **后**\ ，你一共会拥有四个 FastCGI 脚本。


把各实例加入 Ceph 配置文件
--------------------------

在管理节点上，把各例程的配置写入 Ceph 存储集群的配置文件。例如： ::

	...

	[client.radosgw.us-east-1]
	rgw region = us
	rgw region root pool = .us.rgw.root
	rgw zone = us-east
	rgw zone root pool = .us-east.rgw.root
	keyring = /etc/ceph/ceph.client.radosgw.keyring
	rgw dns name = {hostname}
	rgw socket path = /var/run/ceph/$name.sock
	host = {host-name}

	[client.radosgw.us-west-1]
	rgw region = us
	rgw region root pool = .us.rgw.root
	rgw zone = us-west
	rgw zone root pool = .us-west.rgw.root
	keyring = /etc/ceph/ceph.client.radosgw.keyring
	rgw dns name = {hostname}
	rgw socket path = /var/run/ceph/$name.sock
	host = {host-name}


然后，把更新过的 Ceph 配置文件推送到各 :term:`Ceph 节点`\ 如： ::

	ceph-deploy --overwrite-conf config push {node1} {node2} {nodex}


.. note:: 按照以上步骤配置次区域时，需把区域 、存储池和可用区的名字都\
   从 ``us`` 替换为 ``eu`` 。创建主区域和次区域 **后**\ ，你一共\
   会有四条类似配置。


创建区域
-----------

#. 为 ``us`` 区域创建个 infile 配置文件，名为 ``us.json`` 。

   把下列的示例内容复制进文本编辑器，把 ``is_master`` 设置为 ``true`` ，用网关\
   节点的全域名替换 ``{fqdn}`` 。这样，主区域就是 ``us-east`` ，另外 \
   ``zones`` 列表中还会有 ``us-west`` 。详情见\ `配置参考——区域`_ 。 ::

	{ "name": "us",
	  "api_name": "us",
	  "is_master": "true",
	  "endpoints": [
	        "http:\/\/{fqdn}:80\/"],
	  "master_zone": "us-east",
	  "zones": [
	        { "name": "us-east",
	          "endpoints": [
	                "http:\/\/{fqdn}:80\/"],
	          "log_meta": "true",
	          "log_data": "true"},
	        { "name": "us-west",
	          "endpoints": [
	                "http:\/\/{fqdn}:80\/"],
	          "log_meta": "true",
	          "log_data": "true"}],
	  "placement_targets": [
	   {
	     "name": "default-placement",
	     "tags": []
	   }
	  ],
	  "default_placement": "default-placement"}


#. 用刚刚创建的 ``us.json`` infile 文件创建 ``us`` 区域 。 ::

	radosgw-admin region set --infile us.json --name client.radosgw.us-east-1

#. 删除默认区域 （如果有的话）。 ::

	rados -p .us.rgw.root rm region_info.default

#. 把 ``us`` 区域设置为默认区域 。 ::

	radosgw-admin region default --rgw-region=us --name client.radosgw.us-east-1

   一个集群只能有一个默认区域 。

#. 更新区域映射。 ::

	radosgw-admin regionmap update --name client.radosgw.us-east-1


如果为各区域配置不同的 Ceph 存储集群，你应该用 \
``--name client.radosgw-us-west-1`` 选项重复上述的第二、四、五步。也可以从初\
始网关实例导出区域映射，并导入然后更新区域映射。

.. note:: 按照以上步骤配置次区域时，需把 ``us`` 替换为 ``eu`` 。\
   创建主区域和次区域 **后**\ ，你一共会有两个区域。


创建可用区
-----------

#. 为 ``us-east`` 可用区创建名为 ``us-east.json`` 的配置导入文件。

   把以下的示例内容复制到文本编辑器。本配置里的存储池名字用区域名和可\
   用区名作为前缀。关于网关存储池见\ `配置参考——存储池`_\ ，关于可用区\
   请参考\ `配置参考——域`_\ 。 ::

	{ "domain_root": ".us-east.rgw",
	  "control_pool": ".us-east.rgw.control",
	  "gc_pool": ".us-east.rgw.gc",
	  "log_pool": ".us-east.log",
	  "intent_log_pool": ".us-east.intent-log",
	  "usage_log_pool": ".us-east.usage",
	  "user_keys_pool": ".us-east.users",
	  "user_email_pool": ".us-east.users.email",
	  "user_swift_pool": ".us-east.users.swift",
	  "user_uid_pool": ".us-east.users.uid",
	  "system_key": { "access_key": "", "secret_key": ""},
	  "placement_pools": [
	    { "key": "default-placement",
	      "val": { "index_pool": ".us-east.rgw.buckets.index",
	               "data_pool": ".us-east.rgw.buckets",
	               "data_extra_pool": ".us-east.rgw.buckets.extra"}
	    }
	  ]
	}


#. 通过刚创建的配置文件 ``us-east.json`` 将 ``us-east`` 可用区加入东部和西部的存储\
   池，需分别指定其用户名（即 ``--name`` ）。 ::

	radosgw-admin zone set --rgw-zone=us-east --infile us-east.json --name client.radosgw.us-east-1
	radosgw-admin zone set --rgw-zone=us-east --infile us-east.json --name client.radosgw.us-west-1

   重复步骤一，为 ``us-west`` 创建配置导入文件，然后通过刚创建的配置文件 ``us-west.json`` 将 \
   ``us-west`` 可用区加入东部和西部的存储池，需分别指定其用户名（即 ``--name`` ）。 ::

	radosgw-admin zone set --rgw-zone=us-west --infile us-west.json --name client.radosgw.us-east-1
	radosgw-admin zone set --rgw-zone=us-west --infile us-west.json --name client.radosgw.us-west-1


#. 删除默认可用区（如果有的话）。 ::

	rados -p .rgw.root rm zone_info.default


#. 更新区域映射。 ::

	radosgw-admin regionmap update --name client.radosgw.us-east-1

.. note:: 按照以上步骤配置次区域时，需把 ``us-`` 替换为 ``eu-`` 。在各区域 \
   创建完主可用区和次可用区\ **后**\ ，你一共会有四个可用区。


创建可用区用户
--------------

Ceph 对象网关的可用区用户存储在可用区存储池中，所以配置完可用区之后还必须创建可用区用户。\
拷贝各用户的 ``access_key`` 和 ``secret_key`` 字段，以便后续更新域配置信息。 ::

	radosgw-admin user create --uid="us-east" --display-name="Region-US Zone-East" --name client.radosgw.us-east-1 --system --gen-access-key --gen-secret
	radosgw-admin user create --uid="us-west" --display-name="Region-US Zone-West" --name client.radosgw.us-west-1 --system --gen-access-key --gen-secret


.. note:: 按照以上步骤配置次区域时，需把 ``us-`` 替换为 ``eu-`` 。在各区域 \
   创建完主可用区和次可用区\ **后**\ ，你一共会有四个可用区用户。这些用户\
   不同于\ `创建密钥环`_\ 那里创建的用户。


更新可用区配置
--------------

必须用可用区用户更新可用区配置，这样同步代理才能通过可用区的认证。

#. 打开 ``us-east.json`` 可用区配置文件，把创建用户时输出的 ``access_key`` 和 \
   ``secret_key`` 的内容粘帖进配置文件的 ``system_key`` 字段。 ::

	{ "domain_root": ".us-east.rgw",
	  "control_pool": ".us-east.rgw.control",
	  "gc_pool": ".us-east.rgw.gc",
	  "log_pool": ".us-east.log",
	  "intent_log_pool": ".us-east.intent-log",
	  "usage_log_pool": ".us-east.usage",
	  "user_keys_pool": ".us-east.users",
	  "user_email_pool": ".us-east.users.email",
	  "user_swift_pool": ".us-east.users.swift",
	  "user_uid_pool": ".us-east.users.uid",
	  "system_key": {
	    "access_key": "{paste-access_key-here}",
	    "secret_key": "{paste-secret_key-here}"
	  	 },
	  "placement_pools": [
	    { "key": "default-placement",
	      "val": { "index_pool": ".us-east.rgw.buckets.index",
	               "data_pool": ".us-east.rgw.buckets",
	               "data_extra_pool": ".us-east.rgw.buckets.extra"}
	    }
	  ]
	}

#. 保存 ``us-east.json`` 文件，然后更新可用区配置。 ::

	radosgw-admin zone set --rgw-zone=us-east --infile us-east.json --name client.radosgw.us-east-1
	radosgw-admin zone set --rgw-zone=us-east --infile us-east.json --name client.radosgw.us-west-1

#. 重复步骤一更新 ``us-west`` 的配置文件，然后更新可用区配置。 ::

	radosgw-admin zone set --rgw-zone=us-west --infile us-west.json --name client.radosgw.us-east-1
	radosgw-admin zone set --rgw-zone=us-west --infile us-west.json --name client.radosgw.us-west-1


.. note:: 按照以上步骤配置次区域时，需把 ``us-`` 替换为 ``eu-`` 。\
   在各区域创建完主可用区和次可用区\ **后**\ ，你一共会有四个可用区。


重启服务
--------

再次发布 Ceph 配置文件后，我们建议重启 Ceph 存储集群和 Apache 例程。

对于 Ubuntu ，可在各 :term:`Ceph 节点`\ 上执行如下命令： ::

	sudo restart ceph-all

对于 Red Hat/CentOS ，使用下面的命令： ::

	sudo /etc/init.d/ceph restart

为了确保所有组件都重新加载了各自的配置，对于网关实例，我们建议重启 ``apache2`` 服务，\
例如： ::

	sudo service apache2 restart


启动网关实例
------------

启动 ``radosgw`` 服务。 ::

	sudo /etc/init.d/radosgw start

如果你在同一主机上运行了多个实例，那么还必须指定用户名。 ::

	sudo /etc/init.d/radosgw start --name client.radosgw.us-east-1

打开浏览器检查各可用区的连接点，通过其域名发起一个简单的 HTTP 请求应该会得到下面的\
回应：

.. code-block:: xml

   <ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
	   <Owner>
		   <ID>anonymous</ID>
		   <DisplayName/>
	   </Owner>
	   <Buckets/>
   </ListAllMyBucketsResult>


配置次区域
===========

本接提供了配置一个拥有多区域的集群的典型步骤。配置一个跨区域的集群要求\
维护一个全局命名空间，这样存储于不同区域的对象名就不会存在命名空间冲突\
问题了。

本接扩展了\ `配置主区域`_ 里的步骤，同时更改了区域名称及修改了一些\
步骤，详情见下文。


为次区域命名
------------------

配置集群前，规划良好的区域 、可用区和实例名字可帮你更好地管理集群。假设此 \
区域代表欧盟，那我们就引用她的标准缩写。

- European Union: ``eu``

假设两个可用区分别为欧盟的东部和西部，为保持连贯性，命名将采用 \
``{region name}-{zone name}`` 格式，但你可以用自己喜欢的命名规则。

- European Union, East Region: ``eu-east``
- European Union, West Region: ``eu-west``

最后，我们假设每个可用区都配置了至少一个 Ceph 对象网关实例，为保持连贯性，我们将\
按 ``{region name}-{zone name}-{instance}`` 格式命名，但你可以用自己喜欢的命\
名规则。

- European Union Region, Master Zone, Instance 1: ``eu-east-1``
- European Union Region, Secondary Zone, Instance 1: ``eu-west-1``


配置次区域
------------------

再次执行\ `配置主区域`_ 里的典型步骤，不同之处如下：

#. 使用\ `为次区域命名`_\ 而非\ `为主区域命名`_\ 。

#. `创建存储池`_\ ，用 ``eu`` 取代 ``us`` 。

#. `创建密钥环`_\ 及对应密钥，用 ``eu`` 取代 ``us`` 。如果你想用同样的密钥\
   环也可以，只要确保给此区域 （或区域和可用区）创建好密钥就行。

#. `安装 Apache 和 FastCGI`_.

#. `创建数据目录`_\ ，用 ``eu`` 取代 ``us`` 。

#. `创建网关配置`_\ ，把套接字名称里的 ``eu`` 替换为 ``us`` 。

#. `启用配置`_\ 。

#. `添加 FastCGI 脚本`_\ ，把用户名里的 ``eu`` 替换为 ``us`` 。

#. `把各实例加入 Ceph 配置文件`_\ ，把存储池名称里的 ``eu`` 替换为 ``us`` 。

#. `创建区域`_ ，用 ``eu`` 取代 ``us`` 。把 ``is_master`` 设置为 \
   ``false`` ，为保持一致性，在次区域里也创建主区域。 ::

	radosgw-admin region set --infile us.json --name client.radosgw.eu-east-1

#. `创建可用区`_\ ，用 ``eu`` 取代 ``us`` 。确保换成正确的用户名（即 \
   ``--name`` ），这样才会把可用区创建到正确的集群。

#. `更新可用区配置`_\ ，用 ``eu`` 取代 ``us`` 。

#. 在次区域里，创建主区域的各个可用区。 ::

	radosgw-admin zone set --rgw-zone=us-east --infile us-east.json --name client.radosgw.eu-east-1
	radosgw-admin zone set --rgw-zone=us-east --infile us-east.json --name client.radosgw.eu-west-1
	radosgw-admin zone set --rgw-zone=us-west --infile us-west.json --name client.radosgw.eu-east-1
	radosgw-admin zone set --rgw-zone=us-west --infile us-west.json --name client.radosgw.eu-west-1

#. 在主区域里，创建次区域的各个可用区。 ::

	radosgw-admin zone set --rgw-zone=eu-east --infile eu-east.json --name client.radosgw.us-east-1
	radosgw-admin zone set --rgw-zone=eu-east --infile eu-east.json --name client.radosgw.us-west-1
	radosgw-admin zone set --rgw-zone=eu-west --infile eu-west.json --name client.radosgw.us-east-1
	radosgw-admin zone set --rgw-zone=eu-west --infile eu-west.json --name client.radosgw.us-west-1

#. `重启服务`_\ 。

#. `启动网关实例`_\ 。


多站点数据复制
==============

数据同步代理会把主可用区数据复制到次可用区。区域的主可用区是该区域的次可用区\
的数据源,（复制时）自动选择次可用区。

.. image:: ../images/zone-sync.png

配置同步代理需找出源端和目的端的访问密钥、私钥、以及目的 URL 和端口。

你可以用 ``radosgw-admin zone list`` 获取可用区名称列表，用 \
``radosgw-admin zone get`` 得到可用区的访问密钥和私钥。端口可以在\ \
`创建网关配置`_\ 时创建的网关配置文件里找到。

单个实例只需主机名和端口号（假设一个区域或可用区内的所有网关实例都访问\
同一 Ceph 存储集群）；将上述的值添加到配置文件（如 ``cluster-data-sync.conf`` ）\
同时指定一个日志文件给 ``log_file`` 。

例如：

.. code-block:: ini

	src_access_key: {source-access-key}
	src_secret_key: {source-secret-key}
	destination: https://zone-name.fqdn.com:port
	dest_access_key: {destination-access-key}
	dest_secret_key: {destination-secret-key}
	log_file: {log.filename}

实例类似这样：

.. code-block:: ini

	src_access_key: DG8RE354EFPZBICHIAF0
	src_secret_key: i3U0HiRP8CXaBWrcF8bbh6CbsxGYuPPwRkixfFSb
	destination: https://us-west.storage.net:80
	dest_access_key: U60RFI6B08F32T2PD30G
	dest_secret_key: W3HuUor7Gl1Ee93pA2pq2wFk1JMQ7hTrSDecYExl
	log_file: /var/log/radosgw/radosgw-sync-us-east-west.log

要启动数据同步代理，在终端内执行以下命令： ::

	radosgw-agent -c region-data-sync.conf

同步代理运行时，你应该能从输出里看到数据片段正在同步。 ::

	INFO:radosgw_agent.sync:Starting incremental sync
	INFO:radosgw_agent.worker:17910 is processing shard number 0
	INFO:radosgw_agent.worker:shard 0 has 0 entries after ''
	INFO:radosgw_agent.worker:finished processing shard 0
	INFO:radosgw_agent.worker:17910 is processing shard number 1
	INFO:radosgw_agent.sync:1/64 shards processed
	INFO:radosgw_agent.worker:shard 1 has 0 entries after ''
	INFO:radosgw_agent.worker:finished processing shard 1
	INFO:radosgw_agent.sync:2/64 shards processed
	...

.. note:: 每个数据复制源和目的对都要运行代理。


区域间元数据复制
===================

数据同步代理会把主区域内主可用区的元数据复制到次区域的主可用区。元数据包\
含网关用户和桶信息、但不包含桶内的对象——确保跨集群的统一命名空间，主区域 \
内主可用区是次区域内主可用区的数据源，（数据复制时）自动选择次区域的主可用区。

.. image:: ../images/region-sync.png
   :align: center

按照与\ `多站点数据复制`_\ 相同的步骤，指定主区域的主可用区作为源、次 \
区域的主可用区作为目的。启动 ``radosgw-agent`` 时加上 ``--metadata-only`` \
参数，这样它就只会复制元数据。例如： ::

	radosgw-agent -c inter-region-data-sync.conf --metadata-only

完成前述各步骤之后，你就会拥有一套包含两个区域的集群，其内的主区域 \
``us`` 和次区域 ``eu`` 的共享统一的命名空间。


.. _CRUSH Maps: ../../rados/operations/crush-map
.. _安装 Ceph 对象网关: ../../install/install-ceph-gateway
.. _Cephx 管理: ../../rados/operations/authentication/#cephx-administration
.. _Ceph 配置文件: ../../rados/configuration/ceph-conf
.. _配置参考——存储池: ../config-ref#pools
.. _配置参考——区域: ../config-ref#regions
.. _配置参考——可用区: ../config-ref#zones
.. _存储池: ../../rados/operations/pools
.. _简单配置: ../config
