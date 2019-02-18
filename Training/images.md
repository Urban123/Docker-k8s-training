In this lab, we will write a Dockerfile and build a new Docker image.

Chapter Details

| Chapter Goal | Learn how to write Dockerfile and build Docker image |
|---|---|
|   | 1. Build a Docker image  |
|   | 2. Build a multi-container application  |


### Build a Docker image

In the previous section [Docker](./docker.md), we learned how to use Docker images to run Docker containers. Docker images that we used have been downloaded from the Docker Hub, a registry of Docker images. In this section we will create a simple web application from scratch. We will use Flask (http://flask.pocoo.org/), a microframework for Python. Our application for each request will display a random picture from the defined set.

In the next session we will create all necessary files for our application. The code for this application is also available in GitHub: https://github.com/docker/labs/tree/master/beginner/flask-app

#### Create project files
**Step 1** Create a new directory flask-app for our project:
``` 
$ mkdir flask-app
$ cd flask-app
```

**Step 2** In this directory, we will create the following project files:

```
flask-app/
    Dockerfile
    app.py
    requirements.txt
    templates/
        index.html
```

**Step 3** Create a new file app.py with the following content:
```
from flask import Flask, render_template
import random

app = Flask(__name__)

# list of cat images
images = [
    "https://media.giphy.com/media/JIX9t2j0ZTN9S/giphy.gif",
    "https://media.giphy.com/media/mlvseq9yvZhba/giphy.gif",
    "https://media.giphy.com/media/nNxT5qXR02FOM/giphy.gif",
    "https://media.giphy.com/media/yFQ0ywscgobJK/giphy.gif",
    "https://media.giphy.com/media/6uMqzcbWRhoT6/giphy.gif",
    "https://media.giphy.com/media/MWSRkVoNaC30A/giphy.gif",
    "https://media.giphy.com/media/33OrjzUFwkwEg/giphy.gif",
    "https://media.giphy.com/media/l0MYNB04rBb51QNtC/giphy.gif",
    "https://media.giphy.com/media/6VoDJzfRjJNbG/giphy.gif",
    "https://media.giphy.com/media/xBAreNGk5DapO/giphy.gif",
    "https://media.giphy.com/media/8vQSQ3cNXuDGo/giphy.gif"
]

@app.route('/')
def index():
    url = random.choice(images)
    return render_template('index.html', url=url)

if __name__ == "__main__":
    app.run(host="0.0.0.0")
```

**Step 4** Create a new file requirements.txt with the following line:
```
Flask==0.12.4
```
**Step 5** Create a new directory templates and create a new file index.html in this directory with the following content:
```
<html>
  <head>
    <style type="text/css">
      body {
        background: black;
        color: white;
      }
      div.container {
        max-width: 800px;
        margin: 100px auto;
        border: 20px solid white;
        padding: 10px;
        text-align: center;
      }
      h4 {
        text-transform: uppercase;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h4>Cat Gif of the day</h4>
      <img src="{{url}}" />
    </div>
  </body>
</html>
```
**Step 6** Create a new file Dockerfile:
```
# our base image
FROM python:alpine3.7

# install Python modules needed by the Python app
COPY requirements.txt /usr/src/app/
RUN pip install --no-cache-dir -r /usr/src/app/requirements.txt

# copy files required for the app to run
COPY app.py /usr/src/app/
COPY templates/index.html /usr/src/app/templates/

# tell the port number the container should expose
EXPOSE 5000

# run the application
CMD ["python", "/usr/src/app/app.py"]
```

#### Build a Docker image

**Step 1** Now we can build our Docker image. In the command below, replace <user-name> with your user name. For a real image, the user name should be the same as you created when you registered on Docker Hub. Because we will not publish our image, you can use any valid name:
```
$ docker build -t <user-name>/myfirstapp .
Sending build context to Docker daemon 8.192 kB
Step 1 : FROM alpine:latest
...
Successfully built f1277efd5a79
```

**Step 2** Check that your image exists:
```
$ docker images
REPOSITORY     TAG    IMAGE ID     CREATED       SIZE
.../myfirstapp latest f1277efd5a79 6 minutes ago 56.34 MB
```
**Step 3** Run a container in a background and expose a standart HTTP port (80), which is redirected to the container’s port 5000:
```
$ docker run -dp 80:5000 --name myfirstapp <user-name>/myfirstapp
...
```
**Step 5** Use your browser to open the address http://<lab IP> and check that the application works.

**Step 6** Stop the container and remove it:
```
$ docker stop myfirstapp
myfirstapp
$ docker rm myfirstapp
myfirstapp
```

### Build a multi-container application
In this section, will guide you through the process of building a multi-container application. The application code is available at GitHub:
           https://github.com/docker/example-voting-app
           
**Step 1** Clone the existing application from GitHub:
```
$ git clone https://github.com/docker/example-voting-app.git
Cloning into 'example-voting-app'...
...
$ cd example-voting-app
```

**Step 2** Edit the vote/app.py file, change the following lines near the top of the file:
```
option_a = os.getenv('OPTION_A', "Cats")
option_b = os.getenv('OPTION_B', "Dogs")
```

**Step 3** Replace “Cats” and “Dogs” with two options of your choice. For example:
```
option_a = os.getenv('OPTION_A', "Java")
option_b = os.getenv('OPTION_B', "Python")
```

**Step 4** The existing file docker-compose.yml defines several images:
           
* A voting-app container based on a Python image
* A result-app container based on a Node.js image
* A Redis container based on a redis image, to temporarily store the data.
* A worker app based on a dotnet image
* A Postgres container based on a postgres image

Note that three of the containers are built from Dockerfiles, while the other two are images on Docker Hub.

**Step 5** Let’s change the default port to expose. Edit the docker-compose.yml file and find the following lines:
```
...
ports:
  - "5000:80"
...
```
Change 5000 to 80:
```
...
ports:
  - "80:80"
...
```
**Step 6** Install the docker-compose tool:
```
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

**Step 7** Use the docker-compose tool to launch your application:
```
$ docker-compose up -d
```

**Step 8** Check that all containers are running:
```
$ docker ps | grep examplevotingapp
59e355151f56 examplevotingapp_worker "/bin/sh -c 'dotnet W" 2 minutes ago Up 2 minutes                                                     examplevotingapp_worker_1
fd0740ee1525 redis:alpine            "docker-entrypoint.sh" 2 minutes ago Up 2 minutes      0.0.0.0:32772->6379/tcp                        examplevotingapp_redis_1
1fd3b378dd0a examplevotingapp_result "nodemon --debug serv" 2 minutes ago Up 2 minutes      0.0.0.0:5858->5858/tcp, 0.0.0.0:5001->80/tcp   examplevotingapp_result_1
0948fd9ba26c postgres:9.4            "/docker-entrypoint.s" 2 minutes ago Up 2 minutes      5432/tcp                                       examplevotingapp_db_1
52ef44aa1b97 examplevotingapp_vote   "python app.py"        2 minutes ago Up About a minute 0.0.0.0:80->80/tcp                             examplevotingapp_vote_1
```

**Step 9** Use your browser to open the address http://<lab IP> and check that the application works. Then stop the application:
```
$ docker-compose down
Stopping examplevotingapp_worker_1 ... done
Stopping examplevotingapp_redis_1 ... done
Stopping examplevotingapp_result_1 ... done
Stopping examplevotingapp_db_1 ... done
Stopping examplevotingapp_vote_1 ... done
Removing examplevotingapp_worker_1 ... done
Removing examplevotingapp_redis_1 ... done
Removing examplevotingapp_result_1 ... done
Removing examplevotingapp_db_1 ... done
Removing examplevotingapp_vote_1 ... done
Removing network examplevotingapp_default
```