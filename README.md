

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



## Task2: Deploy a simple docker stack which contains two services (use compose file)
### 1. MySQL service with volume.
### 2. Simple app service (you can find some simple Flask apps from communities) which should connect to the above MySQL

* For deploying stack, created a docker-compose.yml with database service and inserted tables contents.

~~~

---

version: '3.8'

services:

  mysql:
    image: mysql:5.6
    networks:
      - flask_net
    environment:
      MYSQL_ROOT_PASSWORD: test123
      MYSQL_DATABASE: helloworld_db
      MYSQL_USER: test_db
      MYSQL_PASSWORD: test_db
    ports:
      - "3306:3306"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.resource == manager
    volumes:
      - type: bind
        source: /root/volume/mysql
        target: /var/lib/mysql

networks:
  flask_net:
    external: true


~~~

~~~
#docker stack deploy -c docker-compose.yml arun

# docker service ls
ID             NAME           MODE         REPLICAS   IMAGE                           PORTS
r0uw00olh3ej   arun_mysql     replicated   1/1        mysql:5.6                       *:3306->3306/tcp

~~~

* For creating the image for Flask app, I have created a Dockerfile, app.py and requirements.txt  in the app directory and build the image for flask app

~~~
 cat Dockerfile
# Use the official Python image from the Docker Hub
FROM python:3.7-slim

# Set the working directory
WORKDIR /app

# Copy the requirements file
COPY requirements.txt .

# Install the dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application files
COPY . .

# Expose the port the app runs on
EXPOSE 5000

# Define environment variable
ENV FLASK_APP=app.py

# Run the application
CMD ["flask", "run", "--host=0.0.0.0"]
~~~

## app.py

~~~cat app.py
from flask import Flask, jsonify
import mysql.connector
import os

app = Flask(__name__)

# MySQL configurations from environment variables
db_config = {
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'host': os.getenv('MYSQL_HOST'),
    'database': os.getenv('MYSQL_DATABASE')
}

def get_db_connection():
    connection = mysql.connector.connect(**db_config)
    return connection

@app.route('/')
def hello_world():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT message FROM messages WHERE id=1")
    result = cursor.fetchone()
    message = result[0] if result else "No message found."
    cursor.close()
    conn.close()
    return jsonify({"message": message})

if __name__ == '__main__':
    app.run(debug=True)

~~~

## requirements.txt
~~~
# cat requirements.txt
Flask
mysql-connector-python
~~~

### Build docker image for flask APP

~~~
# docker image ls
REPOSITORY                 TAG       IMAGE ID       CREATED        SIZE
arunsiby369/test-flask     latest    1bc29eb05c1f   3 hours ago    231MB

~~~

Updated the compose file to deploy the Flask app

~~~
# cat docker-compose.yml
---

version: '3.8'

services:

  mysql:
    image: mysql:5.6
    networks:
      - flask_net
    environment:
      MYSQL_ROOT_PASSWORD: test123
      MYSQL_DATABASE: helloworld_db
      MYSQL_USER: test_db
      MYSQL_PASSWORD: test_db
    ports:
      - "3306:3306"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.resource == manager
    volumes:
      - type: bind
        source: /root/volume/mysql
        target: /var/lib/mysql

  flask:
    image: arunsiby369/test-flask:latest
    networks:
      - flask_net
    depends_on:
      - mysql
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=development
      - MYSQL_USER=test_db
      - MYSQL_PASSWORD=test_db
      - MYSQL_HOST=mysql
      - MYSQL_DATABASE=helloworld_db
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

networks:
  flask_net:
    external: true

~~~

~~~
# docker stack deploy -c docker-compose.yml arun
Creating service arun_mysql
Creating service arun_flask

# docker service ls
ID             NAME           MODE         REPLICAS   IMAGE                           PORTS
7jqrpcegszhu   arun_flask     replicated   1/1        arunsiby369/test-flask:latest   *:5000->5000/tcp
r0uw00olh3ej   arun_mysql     replicated   1/1        mysql:5.6                       *:3306->3306/tcp

~~~

Deployment is completed and it is woking
~~~
root@ip-172-31-83-67:~/flask_mysql_docker# curl http://arunsiby.tech:5000
{
  "message": "This is loading from database"
}
~~~


![task2 1](https://github.com/arunsiby/Test/assets/21075710/5111548f-dc3b-4249-bf47-f6a0296f9b9c)



## Task 3: Deploy another docker stack with traefik service. The request coming to above app should be load balanced through traefik. So, when request hits it first reaches to traefik and then traefik should forward request to docker service.


* Modified the compose file to add traefik service:
  
~~~
# cat docker-compose.yml
---

version: '3.8'

services:

  mysql:
    image: mysql:5.6
    networks:
      - flask_net
    environment:
      MYSQL_ROOT_PASSWORD: test123
      MYSQL_DATABASE: helloworld_db
      MYSQL_USER: test_db
      MYSQL_PASSWORD: test_db
    ports:
      - "3306:3306"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.resource == manager
    volumes:
      - type: bind
        source: /root/volume/mysql
        target: /var/lib/mysql

  flask:
    image: arunsiby369/test-flask:latest
    networks:
      - flask_net
    depends_on:
      - mysql
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=development
      - MYSQL_USER=test_db
      - MYSQL_PASSWORD=test_db
      - MYSQL_HOST=mysql
      - MYSQL_DATABASE=helloworld_db
    deploy:
      replicas: 1
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.flask.rule=Host(`arunsiby.tech`)"
        - "traefik.http.services.flask.loadbalancer.server.port=5000"
      placement:
        constraints:
          - node.role == manager

  traefik:
    image: "traefik:v2.5"
    command:
      - "--api.insecure=true"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
    ports:
      - "80:80"
      - "8080:8080"
    networks:
      - flask_net
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

networks:
  flask_net:
    external: true

~~~


* Deployed traefik stack

~~~
# docker stack deploy -c docker-compose.yml arun
Since --detach=false was not specified, tasks will be created in the background.
In a future release, --detach=false will become the default.
Creating service arun_mysql
Creating service arun_flask
Creating service arun_traefik
~~~

![task3 1](https://github.com/arunsiby/Test/assets/21075710/4000f035-b612-44f1-9d94-8ac523b3731c)


![task3 2](https://github.com/arunsiby/Test/assets/21075710/6c6a3448-8831-45d4-ad66-f48c843d36ef)

![task3 3](https://github.com/arunsiby/Test/assets/21075710/8fb37ebb-ac02-4815-9f13-8f5f7b8dc26e)



