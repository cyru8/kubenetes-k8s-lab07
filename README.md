# kubenetes-k8s-lab07
In this scenario you'll learn how to bootstrap a Kubernetes cluster using Kubeadm.

Kubeadm solves the problem of handling TLS encryption configuration, deploying the core Kubernetes components and ensuring that additional nodes can easily join the cluster. The resulting cluster is secured out of the box via mechanisms such as RBAC.

More details on Kubeadm can be found at https://github.com/kubernetes/kubeadm

######

Step 1 - Initialise Master

Kubeadm has been installed on the nodes. Packages are available for Ubuntu 16.04+, CentOS 7 or HypriotOS v1.0.1+.

The first stage of initialising the cluster is to launch the master node. The master is responsible for running the control plane components, etcd and the API server. Clients will communicate to the API to schedule workloads and manage the state of the cluster.
Task

The following will use CRI-O, a lightweight container runtime for Kubernetes. There is currently a bug meaning that CRI-O needs to be restarted before beginning. Execute the workaround with systemctl restart crio

The command below will initialise the cluster with a known token to simplify the following steps. The command points to an alternative Container Runtime Interface (CRI), in this case, CRI-O.

kubeadm init --cri-socket=/var/run/crio/crio.sock --kubernetes-version $(kubeadm version -o short)

In production, it's recommend to exclude the token causing kubeadm to generate one on your behalf.

Notice, there are no Docker containers running.

docker ps

Instead, everything is managed via CRI-O. The status of which can be explored via crictl

crictl images

crictl ps

Because CRI-O is built for Kubernetes it means there are no Pause containers. This is just one of the many advantages of having a container runtime designed for Kubernetes.

By default, the crictl CLI is configured to communicate with the runtime by the config file at cat /etc/crictl.yaml

######

Step 2 - View Nodes

The cluster has now been initialised. The Master node will manage the cluster, while our one worker node will run our container workloads.
Task

To manage the Kubernetes cluster, the client configuration and certificates are required. This configuration is created when kubeadm initialises the cluster. The command copies the configuration to the users home directory and sets the environment variable for use with the CLI.

sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf

The Kubernetes CLI, known as kubectl, can now use the configuration to access the cluster. For example, the command below will return the two nodes in our cluster.

kubectl get nodes

We can verify the Container Runtime used by describing the node.

kubectl describe node master01  | grep "Container Runtime Version:"

######
Step 3 - Taint Master

In this scenario a single node has been provisioned. Remove the taint to deploy applications to the Master node.

kubectl taint nodes --all node-role.kubernetes.io/master-

Kubernetes has abstract away the underlying Container runtime meaning the commands behave as expected.

kubectl get pods --all-namespaces
######


Step 4 - Deploy Container Networking Interface (CNI)

The Container Network Interface (CNI) defines how the different nodes and their workloads should communicate. There are multiple network providers available, some are listed here.
Task

In this scenario we'll use WeaveWorks. The deployment definition can be viewed at cat /opt/weave-kube.yaml

This can be deployed using kubectl apply.

kubectl apply -f /opt/weave-kube.yaml

Weave will now deploy as a series of Pods on the cluster. The status of this can be viewed using the command kubectl get pod -n kube-system

When installing Weave on your cluster, visit https://www.weave.works/docs/net/latest/kube-addon/ for details.

######

Step 5 - Deploy Pod

The state of the two nodes in the cluster should now be Ready. This means that our deployments can be scheduled and launched.

Using Kubectl, it's possible to deploy pods. Commands are always issued for the Master with each node only responsible for executing the workloads.

The command below create a Pod based on the Docker Image katacoda/docker-http-server.

kubectl run http --image=katacoda/docker-http-server:latest --replicas=1

The status of the Pod creation can be viewed using kubectl get pods

You will notice this is ImageInspectError. This is expected. This is because all images need to be prefixed with the Container Image Registry, such as docker.io for the Docker Hub.

Fix the problem by chaging the image to include the Registry URL.

kubectl set image deployment/http http=docker.io/katacoda/docker-http-server:latest

Because we Tainted the master, once running, you can see the Container running on the master node.

crictl ps | grep docker-http-server

######

Step 6 - Deploy Dashboard

Kubernetes has a web-based dashboard UI giving visibility into the Kubernetes cluster.
Task

Deploy the dashboard yaml with the command kubectl apply -f dashboard.yaml

The dashboard is deployed into the kube-system namespace. View the status of the deployment with kubectl get pods -n kube-system

When the dashboard was deployed, it was assigned a NodePort of 30000. This makes the dashboard available to outside of the cluster and viewable at https://2886795291-30000-kitek02.environments.katacoda.com/

For your cluster, the dashboard yaml definition can be downloaded from https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml.



