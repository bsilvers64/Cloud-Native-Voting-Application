# cloud-native-voting-application

### Technical Stack

   - Frontend: The frontend of this application is built using React and JavaScript (Compiled in container image). It provides a responsive and user-friendly interface for casting votes.

   - Backend and API: The backend of this application is powered by Go (Golang). It serves as the API (Compiled in container image) handling user voting requests. MongoDB is used as the database backend, configured with a replica set for data redundancy and high availability.


### Kubernetes Resources

To deploy and manage this application effectively, we leverage Kubernetes and a variety of its resources:

   - Namespace: Kubernetes namespaces are utilized to create isolated environments for different components of the application, ensuring separation and organization.

   - Secret: Kubernetes secrets store sensitive information, such as API keys or credentials, required by the application securely.

   - Deployment: Kubernetes deployments define how many instances of the application should run and provide instructions for updates and scaling.

   - Service: Kubernetes services ensure that users can access the application by directing incoming traffic to the appropriate instances.

   - StatefulSet: For components requiring statefulness, such as the MongoDB replica set, Kubernetes StatefulSets are employed to maintain order and unique identities.

   - PersistentVolume and PersistentVolumeClaim: These Kubernetes resources manage the storage required for the application, ensuring data persistence and scalability.

### Development walkthrough -

1. For our Voting Application, we will first create a cluster on Amazon EKS. and put an Add-On to it - Amazon EBS CSI Driver. Using this driver allows Kubernetes pods to directly read from and write to EBS volumes.\
   We will also use 2 nodes of t2 medium instances.\
   Workflow for this app -\
   ![](https://lh7-us.googleusercontent.com/m4rL281Di2kd-o4PY1WLKf2TL41JmRdKwy-B_xE9pwt-SVdGpWXo8RHEXC9ySeUWXdJlrrC53cfMENuPdM5HJIGtajkSgS14GI0s0vdec3ij7Oy2VxuKBEOfYOnXBAVeruQlpIZJfkEwannUZn3sj7E)

2. We will use AWS EC2 to create a t2.micro instance and will install kubectl and other tools on that instance. We will add a role EKSaccess to the instance, with a custom policy attached to it.\
   This policy allows certain actions (granting permissions for basic EKS-related operations and Kubernetes API access) ("eks:DescribeCluster", "eks:ListClusters", "eks:DescribeNodegroup", "eks:ListNodegroups", "eks:ListUpdates", "eks:AccessKubernetesApi") on Amazon Elastic Kubernetes Service (EKS) resources.

3. Then in that instance, we download the kubectl binary, make it executable, and copy it to the /usr/local/bin directory. Additionally, update the PATH environment variable to include /usr/local/bin.\
   At the same time, we will add the nodes to our cluster as the pods are in the **nodes**, and without them our pods won’t run.\
   
   Worker machines in Kubernetes are called nodes. Amazon EKS worker nodes run in your AWS account and connect to our cluster's control plane via the cluster API server endpoint.\
   Our node group has a role - “myAmazonEKSNodeRole”. It has these policies attached to it-\
   
   &#x20;**↪** ️ **AmazonEC2ContainerRegistryReadOnly** - grants read-only access to **Amazon Elastic Container Registry (ECR)**. Users or roles with this policy can list and describe ECR repositories, images, and image details. But not pushing or modifying images.\
   
   &#x20;**↪** **AmazonEKS\_CNI\_Policy -** It grants necessary permissions for the networking components of EKS, allowing proper communication between pods in a cluster****\
   
   &#x20; **↪ AmazonEKSWorkerNodePolicy -** It provides permissions required by worker nodes to interact with the EKS control plane, such as describing and listing clusters, describing nodes, and interacting with the EKS API.\
   
   &#x20;**↪ AmazonEBSCSIDriverPolicy -** IAM policy is associated with **Amazon Elastic Block Store** (EBS) CSI (**Container Storage Interface**) driver. CSI is a standardized interface between container orchestrators (like Kubernetes) and storage providers.\
   
   This policy typically grants permissions required by the **EBS CSI driver** to perform its operations, such as attaching and detaching EBS volumes to and from Kubernetes pods.

Instance type is t2.medium

4. After the cluster is ready, we use this command to set the context. setting the context means configuring the **kubectl** command-line tool to use a specific cluster, user, and namespace. The kubectl tool uses a configuration file called **kubeconfig**, which contains information about clusters, users, and contexts.-

|                                                                        |
| ---------------------------------------------------------------------- |
| aws eks update-kubeconfig --name EKS\_CLUSTER\_NAME --region us-west-2 |

This updates the kubeconfig file for an Amazon EKS cluster. The kubeconfig file is used by kubectl to communicate with the Kubernetes cluster.

This command retrieves the necessary information about the specified EKS cluster and updates the kubeconfig file with the cluster configuration. After running this command, we can use kubectl to interact with your EKS cluster.

5. The instance accesses the cluster server via the iam role (EKSaccess) we created, and not any user. So we will give permission to that role to access that cluster. This is done via **ConfigMaps** which is for configuring RBAC (Role-Based Access Control) within a Kubernetes cluster.\
   In Kubernetes, a ConfigMap is an API object that allows you to decouple configuration artifacts from the containerized applications. It provides a way to store configuration data in key-value pairs, and this data can be consumed by pods, containers, or other system components in a cluster.\
   We will use this command to edit the configmap-

|                                                |
| ---------------------------------------------- |
| kubectl edit configmap aws-auth -n kube-system |

And add the role’s arn and name to it.\
    - rolearn: \<arn\_of\_role>

      username: \<role\_name>

      groups:

        - system:masters

6. We will now deploy our mongoDB pods using statefulSets. First, we will create a namespace, on which mongodb will run and it will have 3 replicas, i.e. 3 pods running for mongo. The manifest file will also create persistent storage which is going to create EBS volumes of gb2 type. So once we run this along with the pods, it will also create ebs volumes.
   So applying the mongo-statefulSet.yaml manifest file will create 3 pods and 3 volumes along with PVCs or persistent volume claims to actually use it in our pods. After applying-
   ![](https://lh7-us.googleusercontent.com/euBhfy2zmi9JBvFxUAG3AWjsPPruBGnsa5hN73SSqGHXU9-9q01mWnNnU6nz8i23jvzAh9_uO3Ku4YEsVPzT-S_FC4XKrQ_6Zl2d6V5oL8peqSn3VXiHJWFljr_v5eFknwWwc2R6PY4lYmyJS9qGdkE)
   So mongo-0 should be our primary database, the other two are backups. To do that we need to expose these mongodb pods, and we will do that using [**Service** ](https://kubernetes.io/docs/concepts/services-networking/service/)in k8s. it's a method for exposing a network application that is running as one or more Pods in your cluster. You use a Service to make that set of Pods available on the network so that clients can interact with it.
   We use the Headless-service, where the cluster ip is set to None. We are creating a service but not exposing it to the world. It will get exposed to an API which will get the votes from the frontend and store it into the mongo-db. After applying the mongo-service.yaml -
   ![](https://lh7-us.googleusercontent.com/NMa-w_GuQ-1DKQJKrRVP_y-RN1azZdEGbKSb8ZjnTdYIKETZMi1m2cdG6mvBIxh3U3yoJyBRMqcURQk7cf_waYRWpHod8P9VlkLANGhjQMh6l6R2v8h4gyB4FMdlu__zvp6Zy09NFajVODbcegr52s4)

7. Now we will set the mongo-0 as primary and mongo 1 and 2 as secondary by going into the container, by -

|                                   |
| --------------------------------- |
| kubectl exec -it mongo-0 -- mongo |

And then doing this in the mongo shell-

|                                                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| cat << EOF \| kubectl exec -it mongo-0 -- mongo   rs.initiate();   sleep(2000);   rs.add("mongo-1.mongo:27017");   sleep(2000);   rs.add("mongo-2.mongo:27017");   sleep(2000);   cfg = rs.conf();   cfg.members\[0].host = "mongo-0.mongo:27017";   rs.reconfig(cfg, {force: true});   sleep(5000);   EOF |

8. Now, we create a new DB in mongo-0 -> langdb and populate it with data. It looks like this-
   ![](https://lh7-us.googleusercontent.com/g17qfhz_EntQiYumw-_YR22KApROcYAiYoruzP7WooKjMFliBlNHrv6JFl0JGMFTAQnIgguBBy-YyhRTs1NkdueTzD-0P6_OUARrjazV7YgdssVHJb1-cnBgeOQnVJlCgovf20vVYotX78WB1KNobhQ)\
   We have to create an API so that whenever the user adds their vote to any language, it should be taken from the front-end and be deployed to mongodb and we will be using deployments for that.

9. So we have mongodb with 3 pods, a service and volumes created. We will deploy our API.in our api-deployments.yaml, it will create 2 replicas of API, and hence 2 pods.
   As of now, we have the following -\
   ![](https://lh7-us.googleusercontent.com/3FK6xIYKV7FwP4qErCHG96BXeCOgC6E4h3h_BmNJE0NhlsBkYP-xoc5Z5gRPwurxaoqaxvIp6a9SRjUGqksMENd5uFstdQ5yUUGgY2zkaE3kkMoCmXZo7uRAvST79A2K7sMCVDEnYV0ZNPZ5uqjLP_c)

10. Now we are going to create a load-balancer service. This is via applying the api-service.yaml file or via this command -

|                                                                                           |
| ----------------------------------------------------------------------------------------- |
| kubectl expose deployment api --name=api --type=LoadBalancer --port=80 --target-port=8080 |

![](https://lh7-us.googleusercontent.com/hGt4h9W5oNmX1JYGfgNva7wA-26sRyYqkgomCiOrEZw5KrBK8hHSu2xLNQ-lSS26yx8n_MfOvVtuYOvDGWlVeGHSrRL0RI85s8xACZANsnj8rymGCknBMUu4fy4_83-aHxfKMGcAKpn8ubwp7NWBfmc)

11. Now we will deploy our front-end as well through the frontend-deployment.yaml. 2 replicas means again 2 pods. We will also create a service for this through the frontend-service.yaml to expose the deployment.
    ![](https://lh7-us.googleusercontent.com/c-_1XvyCbg7j7eQRm_oYbkeWMPV-Ay610ZUBDok0_SPPybvLx6i50ypj8tHrRfBHh-JwXzoww8TOo3qs97ac7i2dQucCid9LHLy3UebOBAdI5cc_CqyRHXecCoUS5rYB1p09O18hWwwEEBy-aQ5vCFs)\
    When we hit the front-end load-balancer’s endpoint, we get our app-
    ![](https://lh7-us.googleusercontent.com/ywkxN0q3Ppml9owcqx5prYW-XqIIYD2Fhx9Mx1MM4EtXotlCzMCs_j3zYseIfLI5jX6Oxlit8_85_x869ddN-X-MjnwqnkopfdwmWS_ENRVAuDmY9ovIDWHOD9uV0VtOLAYtklfq6vpSaUcQkABQslU)\

    And querying our db shows same results which means the api is working-

    ![](https://lh7-us.googleusercontent.com/l-eYL6wNyOudnXGJXGH6vtVnOcg1aYs4LJXSLMtv07UU0D37aRoUvYAf6N9pk09m42c8in6nMIbll6iHj-l6YIpW9Idm8hmb9Y3-1G8CKT4EG-7B5IssKStNPr6CTl5ygZdZ3vwO-o8i9ot_sM-50uY)&#x20;
