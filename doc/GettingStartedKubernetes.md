This page explains how to run Vitess on [Kubernetes](http://kubernetes.io).
It also gives the steps to start a Kubernetes cluster with
[Google Container Engine](https://cloud.google.com/container-engine/).

If you already have Kubernetes v0.19+ running in one of the other
[supported platforms](http://kubernetes.io/gettingstarted/),
you can skip the <code>gcloud</code> steps.
The <code>kubectl</code> steps will apply to any Kubernetes cluster.

## Prerequisites

To complete the exercise in this guide, you must locally install Go 1.3+,
Vitess' <code>vtctlclient</code> tool, and Google Cloud SDK. The
following sections explain how to set these up in your environment.

### Install Go 1.3+

You need to install [Go 1.3+](http://golang.org/doc/install) to build the
<code>vtctlclient</code> tool, which issues commands to Vitess.

After installing Go, make sure your <code>GOPATH</code> environment
variable is set to the root of your workspace. The most common setting
is <code>GOPATH=$HOME/go</code>, and the value should identify a
directory to which your non-root user has write access.

In addition, make sure that <code>$GOPATH/bin</code> is included in
your <code>$PATH</code>. More information about setting up a Go
workspace can be found at
[http://golang.org/doc/code.html#Organization]
(http://golang.org/doc/code.html#Organization).

## Build and install <code>vtctlclient</code>

The <code>vtctlclient</code> tool issues commands to Vitess.

``` sh
$ go get github.com/youtube/vitess/go/cmd/vtctlclient
```

This command downloads and builds the Vitess source code at:

``` sh
$GOPATH/src/github.com/youtube/vitess/
```

It also copies the built <code>vtctlclient</code> binary into <code>$GOPATH/bin</code>.

### Set up Google Compute Engine, Container Engine, and Cloud tools

To run Vitess on Kubernetes using Google Compute Engine (GCE),
you must have a GCE account with billing enabled. The instructions
below explain how to enable billing and how to associate a billing
account with a project in the Google Developers Console.

1.  Log in to the Google Developers Console to [enable billing]
    (https://console.developers.google.com/billing).
    1.  Click the **Billing** pane if you are not there already.
    1.  Click **New billing account**
    1.  Assign a name to the billing account -- e.g. "Vitess on
        Kubernetes." Then click **Continue**. You can sign up
        for the [free trial](https://cloud.google.com/free-trial/)
        to avoid any charges.

1.  Create a project in the Google Developers Console that uses
    your billing account:
    1.  In the Google Developers Console, click the **Projects** pane.
    1.  Click the Create Project button.
    1.  Assign a name to your project. Then click the **Create** button.
        Your project should be created and associated with your
        billing account. (If you have multiple billing accounts,
        confirm that the project is associated with the correct account.)
    1.  After creating your project, click **APIs & auth** in the left menu.
    1.  Click **APIs**.
    1.  Find **Google Compute Engine** and **Google Container Engine API**
        and click the **OFF** button for each to enable those two APIs.

1.  Follow the [GCE quickstart guide]
    (https://cloud.google.com/compute/docs/quickstart#setup) to set up
    and test the Google Cloud SDK. You will also set your default project
    ID while completing the quickstart. Start with step 2 in the setup
    process.

    <br>**Note:** During the quickstart, you'll generate an SSH key for
    Google Compute Engine, and you will be prompted to enter a
    passphrase. You will be prompted for that passphrase several times
    when bringing up your Kubernetes cluster later in this guide.

## Start a Kubernetes cluster

1.  Enable or update beta features in the <code>gcloud</code> tool, and install
    the <code>kubectl</code> tool:

    ``` sh
$ gcloud components update beta kubectl
```

    ``` sh
# Check if kubectl is on your PATH:
$ which kubectl
### example output:
# ~/google-cloud-sdk/bin/kubectl
```

    <br>If <code>kubectl</code> isn't on your PATH, you can tell our scripts where
    to find it by setting the <code>KUBECTL</code> environment variable:

    ``` sh
$ export KUBECTL=/example/path/to/google-cloud-sdk/bin/kubectl
```

1.  If you did not complete the [GCE quickstart guide]
    (https://cloud.google.com/compute/docs/quickstart#setup), set
    your default project ID by running the following command.
    Replace <code>PROJECT</code> with the project ID assigned to your
    [Google Developers Console](https://console.developers.google.com/)
    project. You can [find the ID]
    (https://cloud.google.com/compute/docs/overview#projectids)
    by navigating to the **Overview** page for the project
    in the Console.

    ``` sh
$ gcloud config set project PROJECT
```

1.  Set the [zone](https://cloud.google.com/compute/docs/zones#overview)
    that your installation will use:

    ``` sh
$ gcloud config set compute/zone us-central1-b
```

1.  Create a Kubernetes cluster:

    ``` sh
$ gcloud beta container clusters create example --machine-type n1-standard-4 --num-nodes 5
```

1.  While the cluster is starting, you will be prompted several
    times for the passphrase you created while setting up Google
    Compute Engine.

1.  The command's output includes the IP of the Kubernetes master server:

    ```
NAME     ZONE           MASTER_VERSION  MASTER_IP     MACHINE_TYPE   STATUS
example  us-central1-b  0.19.3          1.2.3.4       n1-standard-4  RUNNING
```

    1.  Open /ui on the MASTER_IP in a browser over *HTTPS*
        (e.g. <code>https://1.2.3.4/ui</code>) to see the Kubernetes
        dashboard, where you can monitor nodes, services, pods, etc.

    1.  If you see a <code>ERRCERTAUTHORITY_INVALID</code> error
        indicating that the server's security certificate is not
        trusted by your computer's operating system, click the
        **Advanced** link and then the link to proceed to the URL.

    1.  You should be prompted to enter a username and password to
        access the requested page. You can find the randomly-generated password
        with <code>kubectl config view</code>:

    ``` sh
$ kubectl config view
### example output:
# apiVersion: v1
# clusters:
# - cluster:
#     server: https://1.2.3.4
# ...
# users:
# - name: gke_project_us-central1-b_example
#   user:
#     password: gr8j0Rb11
#     username: admin
```


## Start a Vitess cluster

1.  **Navigate to your local Vitess source code**

    This directory would have been created when you installed
    <code>vtctlclient</code>:

    ``` sh
$ cd $GOPATH/src/github.com/youtube/vitess
```

1.  **Start an etcd cluster:**

    The Vitess [topology service](http://vitess.io/overview/concepts.html#topology-service)
    stores coordination data for all the servers in a Vitess cluster.
    It can store this data in one of several consistent storage systems.
    In this example, we'll use [etcd](https://github.com/coreos/etcd).
    Note that we need our own etcd clusters, separate from the one used by
    Kubernetes itself.

    ``` sh
$ cd examples/kubernetes
vitess/examples/kubernetes$ ./etcd-up.sh
```

    <br>This command creates two clusters. One is for the
    [global cell](http://vitess.io/doc/TopologyService/#global-vs-local),
    and the other is for a
    [local cell](http://vitess.io/overview/concepts.html#cell-(data-center))
    called *test*. You can check the status of the
    [pods](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/pods.md)
    in the cluster by running:

    ``` sh
$ kubectl get pods
```

    <br>It may take a while for each Kubernetes node to download the
    Docker images the first time it needs them. While the images
    are downloading, the pod status will be Pending.<br><br>

    **Note:** In this example, each script that has a name ending in
    <code>-up.sh</code> also has a corresponding <code>-down.sh</code>
    script, which can be used to stop certain components of the
    Vitess cluster without bringing down the whole cluster. For
    example, to tear down the <code>etcd</code> deployment, run:

    ``` sh
vitess/examples/kubernetes$ ./etcd-down.sh
```

1.  **Start vtctld**

    The <code>vtctld</code> server provides a web interface to
    inspect the state of the Vitess cluster. It also accepts RPC
    commands from <code>vtctlclient</code> to modify the cluster.

    ``` sh
vitess/examples/kubernetes$ ./vtctld-up.sh
### example output:
# Creating vtctld service...
# services/vtctld
# Creating vtctld pod...
# pods/vtctld
#
# vtctld address: http://2.3.4.5:30000
```

    <br>To let you access vtctld from outside Kubernetes, the
    <code>vtctld</code> service is created with the <code>type: NodePort</code>
    option. This creates an
    [external service](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md#external-services)
    by exposing a port on each node that forwards to the vtctld service.<br>

    The **vtctld address** printed by `vtctld-up.sh` is thus the external IP
    of one of the nodes, combined with the `nodePort` assigned for vtctld.

1.  **Access vtctld**

    To access the <code>vtctld</code> service from outside
    Kubernetes, you need to open port 30000 in your platform's firewall.
    (If you don't complete this step, the only way to issue commands
    to <code>vtctld</code> would be to SSH into a Kubernetes node
    and install and run <code>vtctlclient</code> there.)<br><br>

    On GCE, you can open the port like this:

    ``` sh
$ gcloud compute firewall-rules create vtctld --allow tcp:30000
```

    <br>You can then access the vtctld web interface at the address printed
    by `vtctld-up.sh` above. In this example, the web UI would be at
    `http://2.3.4.5:30000`.

1.  **Use <code>vtctlclient</code> to call <code>vtctld</code>**

    You can now run <code>vtctlclient</code> locally to issue commands
    to the <code>vtctld</code> service on your Kubernetes cluster.<br><br>

    When you call <code>vtctlclient</code>, the command requires
    the IP address and port for your <code>vtctld</code> service.
    To avoid having to enter that for each command, you can use the
    provided `kvtctl.sh` script, which uses `kubectl` to discover the
    proper address.

    <br>Now, running `kvtctl.sh help` will test your connection to
    <code>vtctld</code> and also list the <code>vtctlclient</code>
    commands that you can use to administer the Vitess cluster.

    ``` sh
# Test the connection to vtctld and list available commands
vitess/examples/kubernetes$ ./kvtctl.sh help
### example output:
# Available commands:
#
# Tablets:
#   InitTablet ...
# ...
```

    ``` sh
# Get usage for a specific command:
vitess/examples/kubernetes$ ./kvtctl.sh help InitTablet
```

    <br>See the [vtctl reference](http://vitess.io/reference/vtctl.html) for a
    web-formatted version of the <code>vtctl help</code> output.

1.  **Start vttablets**

    A Vitess [tablet](http://vitess.io/overview/concepts.html#tablet) is the
    unit of scaling for the database. A tablet consists of the
    <code>vttablet</code> and <code>mysqld</code> processes, running on the same
    host. We enforce this coupling in Kubernetes by putting the respective
    containers for vttablet and mysqld inside a single
    [pod](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/pods.md).

    <br>Run the following script to launch the vttablet pod, which also includes
    mysqld:

    ``` sh
vitess/examples/kubernetes$ ./vttablet-up.sh
### example output:
# Creating test_keyspace.shard-0 pods in cell test...
# Creating pod for tablet test-0000000100...
# pods/vttablet-100
# Creating pod for tablet test-0000000101...
# pods/vttablet-101
# Creating pod for tablet test-0000000102...
# pods/vttablet-102
# Creating pod for tablet test-0000000103...
# pods/vttablet-103
# Creating pod for tablet test-0000000104...
# pods/vttablet-104
```

    <br>Wait until you see all 5 tablets listed in the **Topology** summary page
    for your <code>vtctld</code> instance
    (e.g. <code>http://2.3.4.5:30000/dbtopo</code>).
    This can take some time if a pod was scheduled on a node that needs to
    download the latest Vitess Docker image. You can also check the status of
    the tablets from the command line using `kvtctl.sh`:

    ``` sh
vitess/examples/kubernetes$ ./kvtctl.sh ListAllTablets test
### example output:
# test-0000000100 test_keyspace 0 replica 10.64.1.6:15002 10.64.1.6:3306 []
# test-0000000101 test_keyspace 0 replica 10.64.2.5:15002 10.64.2.5:3306 []
# test-0000000102 test_keyspace 0 replica 10.64.0.7:15002 10.64.0.7:3306 []
# test-0000000103 test_keyspace 0 rdonly 10.64.1.7:15002 10.64.1.7:3306 []
# test-0000000104 test_keyspace 0 rdonly 10.64.2.6:15002 10.64.2.6:3306 []
```

    <br>Note that of the 5 tablets, the first 3 were assigned to be
    **replica** type (for serving live web traffic), while the last 2
    were assigned to be **rdonly** type (for offline processing).
    These allocations can be configured in the `vttablet-up.sh` script.
    See the [tablet](http://vitess.io/overview/concepts.html#tablet)
    reference for more about the available tablet types.<br>

    <br>By bringing up tablets in a previously empty
    [keyspace](http://vitess.io/overview/concepts.html#keyspace),
    you have effectively just created a new
    [shard](http://vitess.io/overview/concepts.html#shard).
    To initialize the keyspace for the new
    shard, call the `kvtctl.sh RebuildKeyspaceGraph` command:

    ``` sh
vitess/examples/kubernetes$ ./kvtctl.sh RebuildKeyspaceGraph test_keyspace
```

    **Note:** Many <code>vtctlclient</code> commands produce no output on
    success.<br><br>

    After this command completes, go back to the <code>vtctld</code>
    UI and click the **Topology** link in the top nav bar. You should see the
    three tablets listed. If you click the address of a tablet, you
    will see the coordination data stored in <code>etcd</code>.<br><br>

    **_Status pages for vttablets_**

    Each <code>vttablet</code> serves a set of HTML status pages on its primary
    port. The <code>vtctld</code> interface provides a **[status]** link for
    each tablet, but the links are actually to internal, per-pod IPs that can
    only be accessed from within Kubernetes.<br><br>

    As a workaround, you can access tablet status pages through the apiserver
    proxy, provided by the Kubernetes master. For example, to see the status
    page for the tablet with ID 100 (recall that our Kubernetes master is
    on public IP 1.2.3.4), you could navigate to:

    ```
https://1.2.3.4/api/v1/proxy/namespaces/default/pods/vttablet-100:15002/debug/status
```

    <br>In the future, we plan to have vtctld directly link through this proxy from
    the **[status]** link.<br><br>

    **_Direct connection to mysqld_**

    Since the <code>mysqld</code> within the <code>vttablet</code> pod is only
    meant to be accessed via vttablet, our default bootstrap settings only allow
    connections from localhost.<br><br>

    If you want to check or manipulate the underlying mysqld, you can issue
    simple queries or commands through `vtctlclient` like this:

    ``` sh
# Send a query to tablet 100 in cell 'test'.
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT VERSION()"
### example output:
# {
#   "Fields": [],
#   "RowsAffected": 1,
#   "InsertId": 0,
#   "Rows": [
#     [
#       "10.0.20-MariaDB-1~wheezy-log"
#     ]
#   ],
#   "Err": null
# }
```

    <br>If you need a truly direct connection to mysqld for bulk operations,
    you can SSH to the Kubernetes node on which the pod is running.
    Then use [docker exec](https://docs.docker.com/reference/commandline/cli/#exec)
    to launch a bash shell inside the mysql container, and connect with the
    <code>mysql</code> command-line client:

    ``` sh
# For example, to connect to the mysql container within the vttablet-100 pod:
$ kubectl get pods | grep vttablet-100
### example output:
# vttablet-100 [...] 10.64.2.9    k8s-example-3c0115e4-node-x6jc  [...]
$ gcloud compute ssh k8s-example-3c0115e4-node-x6jc
k8s-example-3c0115e4-node-x6jc:~$ sudo docker ps | grep vttablet-100
### example output:
# ef40b4ff08fa   vitess/lite:latest [...]  k8s_mysql.16e2a810_vttablet-100[...]
k8s-example-3c0115e4-node-x6jc:~$ sudo docker exec -ti ef40b4ff08fa bash
# Now you're in a shell inside the mysql container.
# We need to tell the mysql client the username and socket file to use.
vttablet-100:/# TERM=ansi mysql -u vt_dba -S /vt/vtdataroot/vt_0000000100/mysql.sock
```

1.  **Elect a master vttablet**

    The tablets all start as slaves by default. In this step, you
    designate one of the tablets to be the master. Vitess
    automatically connects the other slaves' mysqld instances
    so that they start replicating from the master's mysqld.<br><br>

    Since this is the first time the shard has been started,
    the tablets are not already doing any replication, and the
    tablet types are all replica or rdonly. As a
    result, the following command uses the <code>-force</code>
    flag when calling the <code>InitShardMaster</code> command
    to be able to promote one instance to master.


    ``` sh
vitess/examples/kubernetes$ ./kvtctl.sh InitShardMaster -force test_keyspace/0 test-0000000100
```

    <br>**Note:** If you do not include the <code>-force</code> flag 
    here, the command will first check to ensure the provided
    tablet is the only tablet of type master in the shard.
    However, since none of the slaves are masters, and we're not
    replicating at all, that check would fail and the command
    would fail as well.<br><br>

    After running this command, go back to the **Topology** page
    in the <code>vtctld</code> web interface. When you refresh the
    page, you should see that one tablet is the master
    and the others are replica or rdonly.<br><br>

    You can also run this command on the command line to see the
    same data:

    ``` sh
vitess/examples/kubernetes$ ./kvtctl.sh ListAllTablets test
### example output:
# test-0000000100 test_keyspace 0 master 10.64.1.6:15002 10.64.1.6:3306 []
# test-0000000101 test_keyspace 0 replica 10.64.2.5:15002 10.64.2.5:3306 []
# test-0000000102 test_keyspace 0 replica 10.64.0.7:15002 10.64.0.7:3306 []
# test-0000000103 test_keyspace 0 rdonly 10.64.1.7:15002 10.64.1.7:3306 []
# test-0000000104 test_keyspace 0 rdonly 10.64.2.6:15002 10.64.2.6:3306 []
```

1.  **Create a table**

    The <code>vtctlclient</code> tool can be used to apply the database schema
    across all tablets in a keyspace. The following command creates
    the table defined in the <code>create_test_table.sql</code> file:

    ``` sh
# Make sure to run this from the examples/kubernetes dir, so it finds the file.
vitess/examples/kubernetes$ ./kvtctl.sh ApplySchema -sql "$(cat create_test_table.sql)" test_keyspace
```

    <br>The SQL to create the table is shown below:

    ```
CREATE TABLE messages (
  page BIGINT(20) UNSIGNED,
  time_created_ns BIGINT(20) UNSIGNED,
  keyspace_id BIGINT(20) UNSIGNED,
  message VARCHAR(10000),
  PRIMARY KEY (page, time_created_ns)
) ENGINE=InnoDB
```

    <br>You can run this command to confirm that the schema was created
    properly on a given tablet, where <code>test-0000000100</code>
    is a tablet alias as shown by the <code>ListAllTablets</code> command:

    ``` sh
vitess/examples/kubernetes$ ./kvtctl.sh GetSchema test-0000000100
### example output:
# {
#   "DatabaseSchema": "CREATE DATABASE `{{.DatabaseName}}` /*!40100 DEFAULT CHARACTER SET utf8 */",
#   "TableDefinitions": [
#     {
#       "Name": "messages",
#       "Schema": "CREATE TABLE `messages` (\n  `page` bigint(20) unsigned NOT NULL DEFAULT '0',\n  `time_created_ns` bigint(20) unsigned NOT NULL DEFAULT '0',\n  `keyspace_id` bigint(20) unsigned DEFAULT NULL,\n  `message` varchar(10000) DEFAULT NULL,\n  PRIMARY KEY (`page`,`time_created_ns`)\n) ENGINE=InnoDB DEFAULT CHARSET=utf8",
#       "Columns": [
#         "page",
#         "time_created_ns",
#         "keyspace_id",
#         "message"
#       ],
# ...
```

1.  **Start <code>vtgate</code>**

    Vitess uses <code>vtgate</code> to route each client query
    to the correct <code>vttablet</code>. In Kubernetes, a
    <code>vtgate</code> service distributes connections to a pool
    of <code>vtgate</code> pods. The pods are curated by a [replication
    controller](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/replication-controller.md).

    ``` sh
vitess/examples/kubernetes$ ./vtgate-up.sh
```

## Test your instance with a client app

The GuestBook app in the example is ported from the
[Kubernetes GuestBook example](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/guestbook-go).
The server-side code has been rewritten in Python to use Vitess as the storage
engine. The client-side code (HTML/JavaScript) has been modified to support
multiple Guestbook pages, which will be useful to demonstrate Vitess sharding in
a later guide.

``` sh
vitess/examples/kubernetes$ ./guestbook-up.sh
### example output:
# Creating guestbook service...
# services/guestbook
# Creating guestbook replicationcontroller...
# replicationcontrollers/guestbook
```

As with the <code>vtctld</code> service, to access the GuestBook
app from outside Kubernetes, we need to set the <code>type</code> field in the
service definition to something that generates an external service.

In this case, since this is a user-facing frontend, we use
<code>type: LoadBalancer</code>, which tells Kubernetes to create a public
[load balancer](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md#type--loadbalancer)
using the API for whatever platform your Kubernetes cluster is in.

As before, you also need to allow access through your platform's firewall:

``` sh
# For example, to open port 80 in the GCE firewall:
$ gcloud compute firewall-rules create guestbook --allow tcp:80
```

Then, get the external IP of the load balancer for the GuestBook service:

``` sh
$ kubectl get -o yaml service guestbook
### example output:
# apiVersion: v1
# kind: Service
# ...
# status:
#   loadBalancer:
#     ingress:
#     - ip: 3.4.5.6
```

If the status shows <code>loadBalancer: {}</code>, it may just need more time.

Once the pods are running, the GuestBook app should be accessible
from the load balancer's external IP. In the example above, it would be at
<code>http://3.4.5.6</code>.

You can see Vitess' replication capabilities by opening the app in
multiple browser windows, with the same Guestbook page number.
Each new entry is committed to the master database.
In the meantime, JavaScript on the page continuously polls
the app server to retrieve a list of GuestBook entries. The app serves
read-only requests by querying Vitess in 'replica' mode, confirming
that replication is working.

You can also inspect the data stored by the app:

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
### example output:
# {
#   "Fields": [],
#   "RowsAffected": 3,
#   "InsertId": 0,
#   "Rows": [
#     [
#       "42",
#       "1435441767473414912",
#       "9080723075667090943",
#       "First!"
#     ],
#     [
#       "42",
#       "1435441772740816128",
#       "9080723075667090943",
#       "Message 2"
#     ],
#     [
#       "42",
#       "1435441778454107904",
#       "9080723075667090943",
#       "Message 3"
#     ]
#   ],
#   "Err": null
# }
```

The [GuestBook source code]
(https://github.com/youtube/vitess/tree/master/examples/kubernetes/guestbook)
provides more detail about how the app server interacts with Vitess.

## Try Vitess resharding

Now that you have a full Vitess stack running, you may want to go on to the
[Sharding in Kubernetes](http://vitess.io/user-guide/sharding-kubernetes.html)
guide to try out
[dynamic resharding](http://vitess.io/user-guide/sharding.html#resharding).

If so, you can skip the tear-down since the sharding guide picks up right here.
If not, continue to the clean-up steps below.

## Tear down and clean up

Before stopping the Container Engine cluster, you should tear down the Vitess
services. Kubernetes will then take care of cleaning up any entities it created
for those services, like external load balancers.

``` sh
vitess/examples/kubernetes $ ./guestbook-down.sh
vitess/examples/kubernetes $ ./vtgate-down.sh
vitess/examples/kubernetes $ ./vttablet-down.sh
vitess/examples/kubernetes $ ./vtctld-down.sh
vitess/examples/kubernetes $ ./etcd-down.sh
```

Then tear down the Container Engine cluster itself, which will stop the virtual
machines running on Compute Engine:

``` sh
$ gcloud beta container clusters delete example
```

It's also a good idea to remove the firewall rules you created, unless you plan
to use them again soon:

``` sh
$ gcloud compute firewall-rules delete vtctld guestbook
```

## Troubleshooting

If a pod enters the <code>Running</code> state, but the server
doesn't respond as expected, use the <code>kubectl logs</code>
command to check the pod output:

``` sh
# show logs for container 'vttablet' within pod 'vttablet-100'
$ kubectl logs vttablet-100 vttablet

# show logs for container 'mysql' within pod 'vttablet-100'
$ kubectl logs vttablet-100 mysql
```

Post the logs somewhere and send a link to the [Vitess
mailing list](https://groups.google.com/forum/#!forum/vitess)
to get more help.
