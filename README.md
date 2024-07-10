# Machine Test

## Task 1: Install docker and set up docker swarm.

* Created three EC2 instances in AWS and setup one Master node and two worker nodes

* Installed dockeer with the following commands:

~~~
apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt install docker-ce
~~~

![task1 1](https://github.com/arunsiby/Test/assets/21075710/f18b4baa-8d73-471b-8e85-fe9d85a826b8)

* Setup swam cluster in Manager server using:

~~~
docker swarm init
~~~


![task1 2](https://github.com/arunsiby/Test/assets/21075710/6b01b659-a366-4a62-ac06-eb2765d928d2)


* Execute the above worker join token command on the worker nodes to connect them to the manager node

![task1 3](https://github.com/arunsiby/Test/assets/21075710/1e9fb585-79f7-4e3a-b839-c4c25747cc7c)


  
