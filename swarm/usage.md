## 使用
在使用swarm管理集群前，需要把集群中所有的节点的docker daemon的监听方式更改为0.0.0.0:2375,可以有两种方式达到这个目的，第一种是在启动docker daemon的时候指定
> sudo docker -H 0.0.0.0:2375&

第二种方式是直接修改docker的配置文件(以下方式是在ubuntu上面，其他版本的linux上略有不同)
> sudo vim /etc/default/docker

在文件的最后添加下面这句代码：
> DOCKER_OPTS="-H 0.0.0.0:2375 -H unix:///var/run/docker.sock"

需要注意的是，一定要在**所有的**节点上进行的。
修改之后要重启docker
> sudo service docker restart

Docker的集群管理需要使用服务发现(Discovery service backend)功能，Swarm支持以下的几种方式：DockerHub上内置的服务发现功能，本地的文件，etcd，counsel，zookeeper和IP列表，本文会详细讲解前两种方式，其他的用法都是大同小异的。

先说一下本次试验的环境，本次试验包括三台机器，IP地址分别为192.168.1.84,192.168.1.83和192.168.1.124.利用这三台机器组成一个docker集群，其中83这台机器同时充当swarm manager节点。

第一种方式，使用DockerHub上面内置的服务发现功能

第一步：在上面三台机器中的任何一台机器上面执行swarm create命令来获取一个集群标志。这条命令执行完毕后，swarm会前往DockerHub上内置的发现服务中获取一个全球唯一的token，用来标识要管理的集群。
> sudo docker run --rm swarm create

我们在84这台机器上执行这条命令，输出如下：
<pre><code>
rio@084:~$ sudo docker run --rm swarm create
b7625e5a7a2dc7f8c4faacf2b510078e
</code></pre>

可以看到我们返回的token是b7625e5a7a2dc7f8c4faacf2b510078e，每次返回的结果都是不一样的。这个token一定要记住，后面的操作都会用到这个token。

第二步：在**所有**要加入这个集群的节点上面执行swarm join命令，表示要把这台机器加入这个集群当中。在本次试验中，就是要在83,84和124这三台机器上执行下面的这条命令：
<pre><code>
sudo docker run --rm swarm join addr=ip_address:2375 token://token_id
</code></pre>
其中的ip_address换成执行这条命令的机器的IP，token_id换成上一步执行swarm create返回的token。
在83这台机器上面的执行结果如下：
<pre><code>
rio@083:~$ sudo docker run --rm swarm join --addr=192.168.1.83:2375 token://b7625e5a7a2dc7f8c4faacf2b510078e
time="2015-05-19T11:48:03Z" level=info msg="Registering on the discovery service  every 25 seconds..." addr="192.168.1.83:2375" discovery="token://b7625e5a7a2dc7 f8c4faacf2b510078e"
</code></pre>
这条命令不会自动返回，要我们自己执行Ctrl+C返回。

第三步：启动swarm manager
因为我们要使用83这台机器充当swarm manager节点，所以需要在83这台机器上面执行swarm manage命令：
<pre><code>
sudo docker run -d -p 2376:2375 swarm manage token://b7625e5a7a2dc7f8c4faacf2b510078e
</code></pre>
执行结果如下：
<pre><code>
rio@083:~$ sudo docker run -d -p 2376:2375 swarm manage token://b7625e5a7a2dc7f8c4faacf2b510078e
83de3e9149b7a0ef49916d1dbe073e44e8c31c2fcbe98d962a4f85380ef25f76
</code></pre>
这条命令如果执行成功会返回已经启动的swarm的container的ID，此时整个集群已经启动起来了。
现在通过docker ps命令来看下有没有启动成功
<pre><code>
rio@083:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                    NAMES
83de3e9149b7        swarm:latest        "/swarm manage token   4 minutes ago       Up 4 minutes        0.0.0.0:2376->2375/tcp   stupefied_stallman
</code></pre>
可以看到，swarm已经成功启动。
在执行swarm manage这条命令的时候，有几点需要注意的：

1. 这条命令需要在充当swarm manager的机器上执行
2. swarm要以daemon的形式执行
3. 映射的端口可以使任意的除了2375以外的并且是未被占用的端口，但一定不能是2375这个端口，因为2375已经被docker本身给占用了。

集群启动成功以后，现在我们可以在任何一台节点上使用swarm list命令查看集群中的节点了，本实验在124这台机器上执行swarm list命令：
<pre><code>
rio@124:~$ sudo docker run --rm swarm list token://b7625e5a7a2dc7f8c4faacf2b510078e
192.168.1.84:2375
192.168.1.124:2375
192.168.1.83:2375
</code></pre>
输出结果列出的IP地址正是我们使用swarm join命令加入集群的机器的IP地址。
现在我们可以在任何一台安装了docker的机器上面通过命令(命令中要指明swarm manager机器的IP地址)来在集群中运行container了。
本次试验，我们在192.168.1.85这台机器上使用docker info命令来查看集群中的节点的信息。其中info可以换成其他的docker支持的命令。
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 info
Containers: 8
Strategy: spread
Filters: affinity, health, constraint, port, dependency
Nodes: 2
 sclu083: 192.168.1.83:2375
  └ Containers: 1
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 4.054 GiB
 sclu084: 192.168.1.84:2375
  └ Containers: 7
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 4.053 GiB
</code></pre>
结果输出显示这个集群中只有两个节点，IP地址分别是192.168.1.83和192.168.1.84，结果不对呀，我们明明把三台机器加入了这个集群，还有124这一台机器呢？
经过排查，发现是忘了修改124这台机器上面改docker daemon的监听方式，只要按照上面的步骤修改写docker daemon的监听方式就可以了。

在使用这个方法的时候，使用swarm create可能会因为网络的原因会出现类似于下面的这个问题：
<pre><code>
rio@227:~$ sudo docker run --rm swarm create
[sudo] password for rio:
time="2015-05-19T12:59:26Z" level=fatal msg="Post https://discovery-stage.hub.docker.com/v1/clusters: dial tcp: i/o timeout"
</code></pre>

第二种方法：使用文件

第二种方法相对于第一种方法要简单得多，也不会出现类似于上面的问题。

第一步：在swarm manager节点上新建一个文件，把要加入集群的机器啊IP地址和端口号写入文件中，本次试验就是要在83这台机器上面操作：
<pre><code>
rio@083:~$ echo 192.168.1.83:2375 >> cluster
rio@083:~$ echo 192.168.1.84:2375 >> cluster
rio@083:~$ echo 192.168.1.124:2375 >> cluster
rio@083:~$ cat cluster
192.168.1.83:2375
192.168.1.84:2375
192.168.1.124:2375
</code></pre>

第二步：在083这台机器上面执行swarm manage这条命令：
<pre><code>
rio@083:~$ sudo docker run -d -p 2376:2375 -v $(pwd)/cluster:/tmp/cluster swarm manage file:///tmp/cluster
364af1f25b776f99927b8ae26ca8db5a6fe8ab8cc1e4629a5a68b48951f598ad
</code></pre>
使用docker ps来查看有没有启动成功：
<pre><code>
rio@083:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS              PORTS                    NAMES
364af1f25b77        swarm:latest        "/swarm manage file:   About a minute ago   Up About a minute   0.0.0.0:2376->2375/tcp   happy_euclid
</code></pre>
可以看到，此时整个集群已经启动成功。

在使用这条命令的时候需要注意的是注意：这里一定要使用-v命令，因为cluster文件是在本机上面，启动的容器默认是访问不到的，所以要通过-v命令共享。

接下来的就可以在任何一台安装了docker的机器上面通过命令使用集群，同样的，在85这台机器上执行docker info命令查看集群的节点信息：
<pre><code>
rio@s085:~$ sudo docker -H 192.168.1.83:2376 info
Containers: 9
Strategy: spread
Filters: affinity, health, constraint, port, dependency
Nodes: 3
 atsgxxx: 192.168.1.227:2375
  └ Containers: 0
  └ Reserved CPUs: 0 / 4
  └ Reserved Memory: 0 B / 2.052 GiB
 sclu083: 192.168.1.83:2375
  └ Containers: 2
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 4.054 GiB
 sclu084: 192.168.1.84:2375
  └ Containers: 7
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 4.053 GiB
</code></pre>

##swarm调度策略
swarm支持多种调度策略来选择节点。每次在swarm启动container的时候，swarm会根据选择的调度策略来选择节点运行container。目前支持的有:spread,binpack和random.再执行swarm manage的命令起送swarm集群的时候可以通过--strategy参数来指定，默认的是spread。
spread和binpack策略会根据每台节点的可用CPU，内存以及正在运行的containers的数量来给各个节点分级，而random策略，顾名思义，他不会做任何的计算，只是单纯的随机选择一个节点来启动container。这种策略一般只做调试用。
使用spread策略，swarm会选择一个正在运行的container的数量最少的那个节点来运行container。这种情况会导致启动的container会尽可能的分布在不同的机器上运行，这样的好处就是如果有节点坏掉的时候不会损失太多的container。而binpack恰恰相反，这种情况下，swarm会尽可能的把所有的容器放在一台节点上面运行。这种策略会避免容器碎片化，因为他会把未使用的机器分配给更大的container，带来的好处就是swarm会使用最少的节点运行最多的container。
先来演示下--strategy=spread的情况
<pre><code>
rio@083:~$ sudo docker run -d -p 2376:2375 -v $(pwd)/cluster:/tmp/cluster swarm manage --strategy=spread file:///tmp/cluster
7609ac2e463f435c271d17887b7d1db223a5d696bf3f47f86925c781c000cb60
ats@sclu083:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                    NAMES
7609ac2e463f        swarm:latest        "/swarm manage --str   6 seconds ago       Up 5 seconds        0.0.0.0:2376->2375/tcp   focused_babbage
</code></pre>
三台机器除了83运行了swarm之外，其他的都没有运行任何一个container，现在在85这台节点上面在swarm集群上启动一个container
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name node-1 -d -P redis
2553799f1372b432e9b3311b73e327915d996b6b095a30de3c91a47ff06ce981
rio@085:~$ sudo docker -H 192.168.1.83:2376 ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS                  PORTS                          NAMES
2553799f1372        redis:latest        /entrypoint.sh redis   24 minutes ago      Up Less than a second   192.168.1.84:32770->6379/tcp   084/node-1
</code></pre>
在启动一个redis，查看结果
<pre><code>

rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name node-2 -d -P redis
7965a17fb943dc6404e2c14fb8585967e114addca068f233fcaf60c13bcf2190
rio@085:~$ sudo docker -H 192.168.1.83:2376 ps
CONTAINER ID        IMAGE                            COMMAND                CREATED                  STATUS              PORTS                           NAMES
7965a17fb943        redis:latest   /entrypoint.sh redis   Less than a second ago   Up 1 seconds        192.168.1.124:49154->6379/tcp   124/node-2                  
2553799f1372        redis:latest                     /entrypoint.sh redis   29 minutes ago           Up 4 minutes        192.168.1.84:32770->6379/tcp    084/node-1
</code></pre>
再次启动一个redis，查看结果
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name node-3 -d -P redis
65e1ed758b53fbf441433a6cb47d288c51235257cf1bf92e04a63a8079e76bee
rio@085:~$ sudo docker -H 192.168.1.83:2376 ps
CONTAINER ID        IMAGE                            COMMAND                CREATED                  STATUS              PORTS                           NAMES
7965a17fb943        redis:latest                     /entrypoint.sh redis   Less than a second ago   Up 4 minutes        192.168.1.227:49154->6379/tcp   124/node-2
65e1ed758b53        redis:latest                     /entrypoint.sh redis   25 minutes ago           Up 17 seconds       192.168.1.83:32770->6379/tcp    083/node-3
2553799f1372        redis:latest                     /entrypoint.sh redis   33 minutes ago           Up 8 minutes        192.168.1.84:32770->6379/tcp    084/node-1
</code></pre>
可以看到三个container都是分布在不同的节点上面的。

现在来看看binpack策略下的情况。在083上面执行命令：
<pre><code>
rio@083:~$ sudo docker run -d -p 2376:2375 -v $(pwd)/cluster:/tmp/cluster swarm manage --strategy=binpack  file:///tmp/cluster
f1c9affd5a0567870a45a8eae57fec7c78f3825f3a53fd324157011aa0111ac5
</code></pre>
现在在集群中启动三个rediscontainer，查看分布情况：
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name node-1 -d -P redis
18ceefa5e86f06025cf7c15919fa64a417a9d865c27d97a0ab4c7315118e348c
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name node-2 -d -P redis
7e778bde1a99c5cbe4701e06935157a6572fb8093fe21517845f5296c1a91bb2
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name node-3 -d -P redis
2195086965a783f0c2b2f8af65083c770f8bd454d98b7a94d0f670e73eea05f8
rio@085:~$ sudo docker -H 192.168.1.83:2376 ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS                  PORTS                          NAMES
2195086965a7        redis:latest        /entrypoint.sh redis   24 minutes ago      Up Less than a second   192.168.1.83:32773->6379/tcp   083/node-3
7e778bde1a99        redis:latest        /entrypoint.sh redis   24 minutes ago      Up Less than a second   192.168.1.83:32772->6379/tcp   083/node-2
18ceefa5e86f        redis:latest        /entrypoint.sh redis   25 minutes ago      Up 22 seconds           192.168.1.83:32771->6379/tcp   083/node-1
</code></pre>
可以看到，所有的container都是分布在同一个节点上运行的。

##Swarm Filters
swarm的调度器(scheduler)在选择节点运行containers的时候支持几种过滤器 (filter)：Constraint,Affinity,Port,Dependency,Health
这些选项可以在执行swarm manage命令的时候通过--filter选项来设置。
###Constraint Filter
constraint 是一个跟具体节点相关联的键值对，可以看做是每个节点的标签，这个标签可以在启动docker daemon的时候指定，比如
<pre><code>
sudo docker -d --label label_name=label01
</code></pre>
也可以写在docker的配置文件里面，在ubuntu上面是/etc/default/docker
在本次试验中，给083添加标签--label label_name=083,084添加标签--label label_name=084,124添加标签--label label_name=084,
以083为例，打开/etc/default/docker文件，修改DOCKER_OPTS：
<pre><code>
DOCKER_OPTS="-H 0.0.0.0:2375 -H unix:///var/run/docker.sock  --label label_name=083"
</code></pre>
在使用docker run命令启动container的时候使用 -e constarint:key=value的形式，可以指定container运行的节点,比如我们想在84上面启动一个redis container，
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name redis_1 -d -e constraint:label_name==084 redis
fee1b7b9dde13d64690344c1f1a4c3f5556835be46b41b969e4090a083a6382d
</code></pre>
主要，是**两个**等号，不是一个等号，这一点会经常被忽略。
接下来再在084这台机器上启动一个redis container
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name redis_2 -d -e constraint:label_name==084 redis         4968d617d9cd122fc2e17b3bad2f2c3b5812c0f6f51898024a96c4839fa000e1
</code></pre>
然后再在083这台机器上启动另外一个redis container
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name redis_3 -d -e constraint:label_name==083 redis         7786300b8d2232c2335ac6161c715de23f9179d30eb5c7e9c4f920a4f1d39570
</code></pre>
现在来看下执行情况：
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS               NAMES
7786300b8d22        redis:latest        "/entrypoint.sh redi   15 minutes ago      Up 53 seconds       6379/tcp            083/redis_3
4968d617d9cd        redis:latest        "/entrypoint.sh redi   16 minutes ago      Up 2 minutes        6379/tcp            084/redis_2
fee1b7b9dde1        redis:latest        "/entrypoint.sh redi   19 minutes ago      Up 5 minutes        6379/tcp            084/redis_1
</code></pre>
可以看到，执行结果跟预期的一样。
但是如果指定一个不存在的标签的话来运行container会报错。
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run --name redis_0 -d -e constraint:label_name==0 redis
FATA[0000] Error response from daemon: unable to find a node that satisfies label_name==0
</code></pre>

###Affinity Filter
通过使用Affinity Filter，可以让一个container紧挨着另一个container启动，也就是说让两个container在同一个节点上面启动。
现在其中一台机器上面启动一个redis
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376 run -d --name redis redis
ea13eddf667992c5d8296557d3c282dd8484bd262c81e2b5af061cdd6c82158d
rio@085:~$ sudo docker -H 192.168.1.83:2376  ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS                  PORTS               NAMES
ea13eddf6679        redis:latest        /entrypoint.sh redis   24 minutes ago      Up Less than a second   6379/tcp            083/redis
</code></pre>
然后再次启动两台redis
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376  run -d --name redis_1 -e affinity:container==redis redis
bac50c2e955211047a745008fd1086eaa16d7ae4f33c192f50412e8dcd0a14cd
rio@085:~$ sudo docker -H 192.168.1.83:2376  run -d --name redis_1 -e affinity:container==redis redis
bac50c2e955211047a745008fd1086eaa16d7ae4f33c192f50412e8dcd0a14cd
</code></pre>
现在来查看下运行结果,可以看到三个container都是在一台机器上运行
<pre><code>
rio@085:~$ sudo docker -H 192.168.1.83:2376  ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS                  PORTS               NAMES
449ed25ad239        redis:latest        /entrypoint.sh redis   24 minutes ago      Up Less than a second   6379/tcp            083/redis_2
bac50c2e9552        redis:latest        /entrypoint.sh redis   25 minutes ago      Up 10 seconds           6379/tcp            083/redis_1
ea13eddf6679        redis:latest        /entrypoint.sh redis   28 minutes ago      Up 3 minutes            6379/tcp            083/redis
</code></pre>
通过-e affinity:image=image_name命令可以指定只有已经下载了image_name的机器才运行容器
<pre><code>
sudo docker –H 192.168.1.83:2376 run –name redis1 –d –e affinity:image==redis redis 
</code></pre>
redis1这个container只会在已经下载了redis镜像的节点上运行。
<pre><code>
sudo docker -H 192.168.1.83:2376 run -d --name redis -e affinity:image==~redis redis
</code></pre>
这条命令达到的效果是：在有redis镜像的节点上面启动一个名字叫做redis的容器，如果每个节点上面都没有redis容器，就按照默认的策略启动redis容器。
###Port Filter
Port也会被认为是一个唯一的资源
<pre><code>
sudo docker -H 192.168.1.83:2376 run -d -p 80:80 nginx
</code></pre>
执行完这条命令，之后任何使用80端口的容器都是启动失败。



