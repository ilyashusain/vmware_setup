# Docker Pipeline

In this guide we will look at my setup for a reverse proxy for our go server using nginx.

# Preliminaries

Create a projects folder named own-project, and inside it create two folders called jenkins and nginx, and touch a docker-compose.yml file. We will now take a look at what these two folders contain, and the yml file, and see how they all interact with the bigger picture.

# Jenkins folder

Inside the Jenkins folder we have the Dockerfile, which allows us to build the specified images e.g. prebuilt Jenkins images.

The docker-run deploys the container.

The init.groovy creates all of the defaults for the Jenkins user e.g. username=admin, password=admin. This can clearly be seen by running vim on the init.groovy file.

The plugins.txt contains the plugins for Jenkins, so we don’t have to wait to install plugins on Jenkins startup like we usually do.

The jobs folder contain what is essentially our server. When we run Jenkins, the above server will be the first thing we see. Next, let’s see what’s inside the go-server folder:

# go-server folder

The most important file in the go-server folder is the config.xml, what it contains is what will be within the build for the go-server on Jenkins. Thus, we automatically created a build for Jenkins. Below we will look at the config file by considering its components separately:

The ```BRANCH_NAME``` is specified first. Git branch will list all the branches, we then grep the active branch with a grep \*. Remember, in bash, the active branch in git is specified by a *:

  master
* test

In this case, the active branch is test and we grep it. We then cut all other branches with cut -d ' ' -f2. Our BRANCH_NAME is now test.

We then define the project, container, image and docker variables for ease:

```bash
# stop container if it's running
if $docker ps | grep $container; then
        $docker stop $container
fi

# delete container if it exists
if $docker ps -a | grep $container; then
        $docker rm $container
fi

# remove image if it exists
if $docker images -q $image; then
        $docker rmi $image
fi
```

The above block of text is self-explanatory. If the container is running, stop it. If container exists, delete it. If image exists, remove it.

```bash
$docker build -t $image .
$docker run -d --name $container $image
$docker network connect own-project_default $container
echo "location /${project}/${BRANCH_NAME}/ {" > ${container}.conf
echo -e "\t proxy_pass http://${container}:8080/;" >> ${container}.conf
echo "}" >> ${container}.conf
$docker cp ${container}.conf nginx:/etc/nginx/conf.d/${container}.conf
$docker exec nginx nginx -s reload
```

We have the usual build and run commands for docker. Notice that in the run command, we do not need to specify the ports, this is done by nginx for us. Remember, normally we would have to run, for example:

```$docker run -d –p 8080:8080 --name $container $image```

specifying the ports, but thanks to nginx which does it for us we can instead run

```$docker run -d --name $container $image.```

The docker network connect command will connect to the docker network. Enter docker network ls in your vm to see the list of available networks, then enter the required network, this is usually the folder where you created your Jenkins and nginx folders followed by _default suffix, in our case this will be own-project_default.

We then echo the location, and echo the proxy_pass. The proxy_pass allows us to run Jenkins through nginx (see: nginx folder).

# nginx folder:

Now we will take a look at the nginx folder. Let’s look at its contents:

```bash
-rw-rw-r--. 1 ihusain1994 ihusain1994  90 Nov 21 14:17 Dockerfile
-rw-rw-r--. 1 ihusain1994 ihusain1994 137 Nov 21 13:31 nginx.conf
```

We again have a Dockerfile, let’s look at its contents:

```bash
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
RUN rm -f /etc/nginx/conf.d/default.conf
```

We can see that it uses the nginx image with the FROM prefix.

It copies the nginx.conf into the /etc/nginx with the COPY prefix.

It also removes any defaults from the /etc/nginx with the RUN prefix. Now let’s look at the actual nginx.conf:

```bash
events {}
http {
        server {
                listen 80;
                include /etc/nginx/conf.d/*.conf;
                location / {
                        proxy_pass http://jenkins:8080/;
                }
        }
}
```

The events{} is a requirement for nginx. We won’t focus on what it actually means at the moment for the sake of simplicity.

We can see that we’re listening on port 80, this is set to default but stated for your own clarity.

The include command allows you to retrieve every config file, instead of just specifically one.

The location command is the key feature of the nginx, this is what allows us to connect to Jenkins without specifying the port number in the docker run command (we discussed this earlier).

# docker-compose.yml:

The yml file is straightforward and easy to read, that is one of the positive attributes of .yml files. See our .yml file below:

```bash
version: '3'
services:
  jenkins:
    build: jenkins
    container_name: jenkins
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./jenkins/jobs:/var/jenkins_home/jobs:Z"
  nginx:
    build: nginx
    container_name: nginx
    ports:
      - "80:80"
```

For Jenkins, we have the build and container name. For volumes we have two lines of code:

```- "/var/run/docker.sock:/var/run/docker.sock"```

Will allow docker to run within Jenkins. The next line: 

```- "./jenkins/jobs:/var/jenkins_home/jobs:Z"```

Retrieves your jobs folder within the jenkins folder.

For nginx we also have a build and container name. Ports are set to 80:80.


# Conclusions:

We created a Jenkins and nginx folder, and understood the functionality each of these have in creating our go server by considering their respective components. We also saw how the .yml file configures Jenkins and nginx.
