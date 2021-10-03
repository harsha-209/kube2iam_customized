# kube2iam_customized

As per my understanding first setup a kubernetes cluster in 2 vm's or EC2 instances

steps to follow me:

1. attach a iam role to worker node server (node server(like ec2 or vm)


{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}


attach required permissions for this above iam roles for workernode machines

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "s3-object-lambda:*"
            ],
            "Resource": "*"
        }
    ]
}



2. now attach above iam role to workernode machines



4. now create a role for pod follow below steps:



  i) create a role with ec2 assume role
  ii) now edit that trust relationship with this below role

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "YOUR_NODE_ROLE"  ###  paste here above created role arn here 
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
4. after editing or  creating a above role
5. now attach required permission for your above role
6. {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "s3-object-lambda:*"
            ],
            "Resource": "*"
        }
    ]
}

7. now we completed creating two roles ,1. for workernode and 2 . for pods

8. now we have to create a service account for permissions for other pods in cluster 
9. copy that above serviceaccount yaml file and run


kubectl apply -f service_account.yaml

after creating above service account now run that kube2iam service 



now copy  kube2iam.yaml script and run 

kubectl apply -f kube2iam.yaml file



10. now check it where above kube2iam service is running or not , if everything ok now launch you pod with pod role follow below pod yaml file
11. kubectl apply -f podaws.yaml file




kubectl 



















you can refer this below link
https://www.bluematador.com/blog/iam-access-in-kubernetes-installing-kube2iam-in-production


 

Step 1: Create IAM Roles



The first step to using kube2iam is to create IAM roles for your pods. The way kube2iam works is that each node in your cluster will need an IAM policy attached which allows it to assume the roles for your pods.

Create an IAM policy for your nodes and attach it to the the role your Kubernetes nodes run on

Policy:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Resource": “*”
    }
  ]
}
 
Next create roles for each pod. Each role will need a policy that has only the permissions that the pod needs to perform its function e.g. listing s3 objects, writing to DynamoDB, reading from SQS, etc. For each role you create, you need to update the assume role policy so that your nodes can assume the role. Replace YOUR_NODE_ROLE with the arn of the role your Kubernetes nodes run with.

Assume role policy:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "YOUR_NODE_ROLE"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
 

Step 2: Add Annotations to Pods





The next step is to annotate your pods with the role they should use. Just add an annotation in the pod metadata spec, and kube2iam will use that role when authenticating with IAM for the pod. Kube2iam will automatically detect the base arn for your role when configured to do so, but you can also specify a full arn (beginning with arn:aws:iam) if you need to assume roles in other AWS accounts. The kube2iam documentation has several examples of annotating different pod controllers.

annotations:
   iam.amazonaws.com/role: MY_ROLE_NAME
 

Step 3: Deploy Kube2iam






Now you are ready to deploy kube2iam. You can reference the kube2iam github repo to get examples for running in EKS or OpenShift, but I will also go over the general deployment method here. The first step is to set up RBAC:

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube2iam
  namespace: kube-system
---
apiVersion: v1
items:
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kube2iam
    rules:
      - apiGroups: [""]
        resources: ["namespaces","pods"]
        verbs: ["get","watch","list"]
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kube2iam
    subjects:
    - kind: ServiceAccount
      name: kube2iam
      namespace: kube-system
    roleRef:
      kind: ClusterRole
      name: kube2iam
      apiGroup: rbac.authorization.k8s.io
kind: List
 
Since kube2iam modifies the iptables on your Kubernetes nodes to hijack traffic to the EC2 metadata service, I recommend adding a new node to your cluster that is tainted so you can do a controlled test to make sure everything is set up correctly without affecting your production pods. Add a node to your cluster and then taint it so other pods will not run on it:

kubectl taint nodes NODE_NAME kube2iam=kube2iam:NoSchedule 


Now we can configure the agent to run only on that node. Add the nodeName key to the pod spec with the name of your new node and then add the tolerations so it will run on that node. Set the image to a tagged release of kube2iam instead of using latest. You also need to set the --host-interface command arg to match your CNI. The kube2iam page has a full list of supported values for this. I also recommend setting the --auto-discover-base-arn and --auto-discover-default-role flags to make configuring and migrating easier. The --use-regional-sts-endpoint is great if your cluster is in a single region, but you must also set the AWS_REGION environment variable for it to work. All together, your config should look something like this:

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube2iam
  namespace: kube-system
  labels:
    app: kube2iam
spec:
  selector:
    matchLabels:
      name: kube2iam
  template:
    metadata:
      labels:
        name: kube2iam
    spec:
      nodeName: NEW_NODE_NAME
      tolerations:
       - key: kube2iam
         value: kube2iam
         effect: NoSchedule
      serviceAccountName: kube2iam
      hostNetwork: true
      containers:
        - image: jtblin/kube2iam:0.10.6
          imagePullPolicy: Always
          name: kube2iam
          args:
            - "--app-port=8181"
            - "--auto-discover-base-arn"
            - "--iptables=true"
            - "--host-ip=$(HOST_IP)"
            - "--host-interface=weave"
            - "--use-regional-sts-endpoint"
            - "--auto-discover-default-role"
            - "--log-level=info"
          env:
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: AWS_REGION
              value: "us-east-1"
          ports:
            - containerPort: 8181
              hostPort: 8181
              name: http
          securityContext:
            privileged: true
 
Now you can create the agent and verify that only a single agent is running on your new node. There should be no change to your pods running on other nodes.

 


Step 4: Test





At this point, you will want to get started with testing that everything works. You can do this by deploying a pod to the quarantine node and then using the AWS CLI to test access to resources in your pod. While you are doing this, check the logs of the kube2iam agent to debug any issues you encounter. Here’s an example of a deployment where you can specify a role and then test access:

apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: aws-iam-tester
  labels:
    app: aws-iam-tester
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: aws-iam-tester
  template:
    metadata:
      labels:
        app: aws-iam-tester
      annotations:
        iam.amazonaws.com/role: TEST_ROLE_NAME
    spec:
      nodeSelector:
        kubernetes.io/role: node
      nodeName: NEW_NODE_NAME
      tolerations:
       - key: kube2iam
         value: kube2iam
         effect: NoSchedule
      containers:
      - name: aws-iam-tester
        image: garland/aws-cli-docker:latest
        imagePullPolicy: Always
        command:
          - /bin/sleep
        args:
          - "3600"
        env:
          - name: AWS_DEFAULT_REGION
            value: us-east-1
 
The pod will exit after an hour, and you can get use kubectl to get a TTY to the pod 

kubectl exec -it POD_NAME /bin/sh 
Once you are satisfied that your roles work, and that the kube2iam agent is correctly set up, you can then deploy the agent to every node.

 

Step 5: Full Kube2iam Deployment






Remove the nodeName key and kube2iam:kube2iam tolerations from your kube2iam DaemonSet to allow it to run on every node. Once it is installed on each node, you should roll out an update to critical pods to ensure that those pods begin using kube2iam for authentication immediately. Other pods that were using the node role to authenticate will begin going to kube2iam when their temporary credentials expire (usually about an hour). Check your application logs and the kube2iam logs for any IAM errors.

Once everything is running correctly, you can remove the quarantine node that was added earlier.

If you encounter issues, you can delete the agent from all nodes, but it will not automatically clean up the iptables rule it created. This will cause all of the calls to EC2 metadata to go nowhere. You will have to ssh into each node individually and remove the iptable rule yourself.

First list the iptable rules to find the one set up by kube2iam

sudo iptables -t nat -S PREROUTING | grep 169.254.169.254 
The output should be similar to

-A PREROUTING -d 169.254.169.254/32 -i weave -p tcp -m tcp --dport 80 -j DNAT --to-destination 10.0.101.101:8181 
You may see multiple results if you deployed the agent with different --host-interface options on accident. You can delete them one at a time. To delete a rule, use the -D option of iptables and specify the entire line of output from above after -A. For example:

sudo iptables -t nat -D PREROUTING -d 169.254.169.254/32 -i weave -p tcp -m tcp --dport 80 -j DNAT --to-destination 10.0.101.101:8181 
As this is done on each node, EC2 metadata requests will no longer go to kube2iam.

 

Conclusion
You should now have kube2iam running in your production cluster with separate IAM roles for your pods.

If you are interested in another solution, check out the kiam installation post I wrote previously. It is much more involved, and uses a different approach to the IAM issue. I also have my eye on kube-aws-iam-controller as an alternative solution, but the project is still very young and not suitable for production. You can also check out the first and second posts in this series for an overview of the AWS IAM problem in Kubernetes, and an in-depth comparison of kiam and kube2iam as solutions.
