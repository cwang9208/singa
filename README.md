#Apache SINGA

Distributed deep learning system

<a name="Dependencies"</a>
##Dependencies
The current code depends on the following external libraries:

  * `glog` (New BSD)
  * `google-protobuf` (New BSD)
  * `openblas` (New BSD)

###Optional dependencies
For advanced features, the following libraries are needed:

  * `zeromq` (LGPLv3 + static link exception),`czmq` (Mozilla Public License Version 2.0) and `zookeeper` (Apache 2.0), for distributed training with multiple processes. Compile SINGA with `--enable-dist`
  * `Apache Mesos` (Apache 2.0)

We have tested SINGA on Ubuntu 12.04, Ubuntu 14.01 and CentOS 6.
You can install all dependencies (including optional dependencies) into `$PREFIX` folder by

    ./thirdparty/install.sh all $PREFIX

If `$PREFIX` is not a system path, please export the following
variables to continue the building instructions,

    $ export LD_LIBRARY_PATH=$PREFIX/lib:$LD_LIBRARY_PATH
    $ export CPLUS_INCLUDE_PATH=$PREFIX/include:$CPLUS_INCLUDE_PATH
    $ export LIBRARY_PATH=$PREFIX/lib:$LIBRARY_PATH
    $ export PATH=$PREFIX/bin:$PATH


##Building SINGA

Please make sure you have `g++ >= 4.8.1` before building SINGA.

    $ ./autogen.sh
    $ ./configure
    $ make

##Training on a single node

Let us train the CNN model over the
CIFAR-10 dataset without parallelism as an example. More details about this example are available
at [CNN example](http://singa.incubator.apache.org/docs/cnn).

First, download the dataset and create data shards:

    $ cd examples/cifar10/
    $ cp Makefile.example Makefile
    $ make download
    $ make create

Next, start the training:

    $ cd ../../
    $ ./singa -conf examples/cifar10/job.conf

You can list the current running jobs by,

    ./bin/singa-console.sh list

Jobs can be killed by,

    ./bin/singa-console.sh kill JOB_ID

##Training in a cluster

###SINGA Architecture
####Logical Architecture

![logical](https://github.com/wangchenghku/singa/blob/master/.resources/logical.png)

The architecture consists of multiple server groups and worker groups:
* Typically, a server group contains a number of servers, and each server manages a partition of model parameters.
* A worker group trains a complete model replica against a partition of the training dataset, and is responsible for computing parameter gradients.

###Distributed Training Framework
####Cluster Topology Configuration

The mostly used fields are as follows:
* `nworkers_per_group` and `nworkers_per_procs`: decide the partitioning of worker side ParamShard.
* `nservers_per_group` and `nservers_per_procs`: decide the partitioning of server side ParamShard.

####Different Training Frameworks

![frameworks](https://github.com/wangchenghku/singa/blob/master/.resources/frameworks.png)

\###Downpour

    cluster {
        nworker_groups: 2
        nserver_groups: 1
        nworkers_per_group: 2
        nservers_per_group: 2

        # nworker_per_procs: 1
        # Every process would then create only one worker thread.
        # Consequently, the workers would be created in different processes (i.e., nodes).
    }

\###AllReduce

	cluster {
        ...
        server_worker_separate: false
        # servers and workers in different processes?
        # default = false
	}

###Training
Update the configuration file *examples/cifar10/job.conf* with:

    nworkers_per_group: 2
    nworker_per_procs: 1

The *hostfile* must be provided under *SINGA_ROOT/conf/* specifying the nodes in the cluster, e.g.,

    192.168.0.1
    192.168.0.2

SINGA uses zookeeper to coordinate the training, and uses ZeroMQ for transferring messages. After installing zookeeper and ZeroMQ, you need to configure SINGA with `--enable-dist` before compiling. Please make sure the zookeeper service is started before running SINGA.

    vi conf/singa.conf
    # SINGA uses zookeeper to coordinate the training
    # deploy zookeeper on a separate machine
    zookeeper_host: "192.168.0.3:2138"

    # set if you want to change log directory
    log_dir: "/tmp/singa-log/"

Add the environment variables that cannot be recognized after ssh in *conf/profile* file

The running command is :

    ./bin/singa-run.sh -conf examples/cifar10/job.conf

##Troubleshooting

###Building OpenBLAS in QEMU/KVM
* type "make" to detect the CPU automatically. or
* type "make TARGET=xxx" to set target CPU, e.g. "make TARGET=NEHALEM". The full target list is in file TargetList.txt.

By default, QEMU reports the CPU as "QEMU Virtual CPU version 2.5+", which OpenBLAS recognizes as PENTIUM2. Depending on the exact combination of CPU features the hypervisor choses to expose, this may not correspond to any CPU that exists, and OpenBLAS will error when trying to build. To fix this, pass `-cpu host` to QEMU, or another CPU model.

###Passwordless SSH logins
**Generate your key pair** - One of the login modes of ssh is to use a SSH key pair.

    ssh-keygen -t dsa -b 2048

**Copy public key to remote machine** - Once you made your key pair, you should copy your public key to the remote machine preferably using an encrypted method such as scp and add it to your .ssh/authorized_keys file. You can do this with a single command.

    cat ~/.ssh/id_dsa.pub | ssh user@remote.machine.com 'cat >> .ssh/authorized_keys'
    # If you need to make a .ssh directory on the remote machine
    cat ~/.ssh/id_dsa.pub | ssh user@remote.machine.com 'mkdir .ssh; cat >> .ssh/authorized_keys'
