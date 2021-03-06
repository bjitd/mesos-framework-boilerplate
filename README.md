# mesos-framework-boilerplate

A boilerplate for developing Mesos frameworks with JavaScript, based on [mesos-framework](https://github.com/tobilg/mesos-framework). It includes a framework scheduler example, completed by an UI and support for a highly configureable framework use that does not need any added code. 

## Intro

The intention of this project is to lower the barrier for Mesos beginners to write a custom framework. That's also the reason why JavaScript was chosen as an implementation language.

[mesos-framework](https://github.com/tobilg/mesos-framework) doesn't currently support HA schedulers, but this will probably arrive in the close future. This means that only single instances of the newly created schedulers can be run, however, it does persist information in Zookeeper across scheduler restarts.  

It supports authentication (via GitLab or Google), restarts, pluggable modules (just write them and put them in a directory named "module-*"), health checks, multiple tasks, colocation prevention support, health check and leader indications (need module code).

## Usage

You can use and customize this project by doing a 

```bash
git clone https://github.com/tobilg/mesos-framework-boilerplate.git
```

In addition, you can customize the marathon task environment variables to run whatever you like as a framework.

### Tools
You'll also need the following tools installed on you machine to be able to develop you own Mesos framework with this boilerplate:

* Node.js >= 4
* NPM >= 2
* Bower

### Installation of dependencies

You'll need to do a 

```bash
npm install && bower install
```

on the command-line in the project's root directory to install the frontend and backend dependencies.

See the chapter [customization](#scheduler-customization) for further info on how to create your own framework scheduler.

## UI

The UI was initially taken from the excellent [mesos/elasticsearch](https://github.com/mesos/elasticsearch) framework, which uses [rdash-angular](https://github.com/rdash/rdash-angular) itself. It then was adapted to use the Node.js backend webservices, which the frontend consumes via Angular services.

If you want to change the UI styling, you can edit the `public/stylesheets/style.css` file according to your needs, in addition, the FRAMEWORK_NAME_BACKGROUND environment variable is available to set the framework title row color.

## Backend

The backend was developed in Node.js, and is quite straight forward. Nothing fancy, just some plain [Express.js](http://www.expressjs.com) services which can be found in `routes/api.js` and `lib/baseApi.js`.

It provides the following endpoints:

```
GET /framework/configuration              Returns the started framework configuration as JSON, authentication-aware.
GET /framework/info                       Returns the information about the framework from the `package.json` file.
GET /framework/stats                      Returns the stats for each task type, for usage in the Dashboard.
GET /framework/restart                    Restarts the scheduler.
GET /tasks/launched                       Returns the launched (running) tasks.
GET /tasks/types                          Returns the task types, along with some basic stats (with `runningInstances` and `allowScaling`).
PUT /tasks/types/:type/scale/:instances   Used to scale the task types (`type`) to (`instances`) instances.
POST /tasks/:task/restart                 Used to restart a task, by task ID, a module can add a health check and set the restart helper to use it as a check of a completed restart.
POST /tasks/rollingRestart                Used for rolling restart of all running tasks, a module can add a health check and set the restart helper to use it as a check of a completed restart.
POST /tasks/killAll                       Used to kill all tasks under the framework and start them again, not as a rolling restart.
POST /tasks/types/:type/killAll           Used to kill all tasks of a type under the framework and start them again, not as a rolling restart.
GET /logs                                 Returns the current log output as text.
GET /health                               Returns http status 200 as long the application is running and a heartbeat from Mesos has been sent in the last 60 seconds. Used for Marathon health checks.
```

## Scheduler customization

The [mesos-framework](https://github.com/tobilg/mesos-framework) documentation applies for all the possible usages of the Scheduler (and possible Executor) classes. Also, have a look at the [examples](https://github.com/tobilg/mesos-framework/tree/master/examples)!

The following sub-chapters are a walk-through of the scheduler configuration for this project.

### The ContainerInfo object

The `ContainerInfo` object is the representation of the [Mesos.ContainerInfo](https://github.com/apache/mesos/blob/c6e9ce16850f69fda719d4e32be3f2a2e1d80387/include/mesos/v1/mesos.proto#L1744) protocol buffer.

You can configure the settings for your Docker image, like the image name and the networking.

```javascript
// The container information object to be used
var ContainerInfo = new Mesos.ContainerInfo(
    Mesos.ContainerInfo.Type.DOCKER, // Type
    null, // Volumes
    null, // Hostname
    new Mesos.ContainerInfo.DockerInfo(
        "mesoshq/flink:0.1.1", // Image
        Mesos.ContainerInfo.DockerInfo.Network.HOST, // Network
        null,  // PortMappings
        false, // Privileged
        null,  // Parameters
        true, // forcePullImage
        null   // Volume Driver
    )
);
```

### The frameworkTasks object
 
The `frameworkTasks` object is a map object, which contains the different kinds of workloads ("task types") the scheduler should deploy on the Mesos cluster. In this example, there are two different workloads, the `jobmanagers` and the `taskmanagers`.

You can define the `priority` in which the task types should be started (lower number is a higher priority). Also, the `instances` property define how many instances of this task shall be started.

The [Mesos.Environment](https://github.com/apache/mesos/blob/c6e9ce16850f69fda719d4e32be3f2a2e1d80387/include/mesos/v1/mesos.proto#L1434) protobuf can be used to define environment variables that should be used when starting the Docker image.

```javascript
// The framework tasks
var frameworkTasks = {
    "jobmanagers": {
        "priority": 1,
        "instances": 3,
        "executorInfo": null, // Can take a Mesos.ExecutorInfo object
        "containerInfo": ContainerInfo, // Mesos.ContainerInfo object
        "commandInfo": new Mesos.CommandInfo( // Strangely, this is needed, even when specifying ContainerInfo...
            null, // URI
            new Mesos.Environment([
                new Mesos.Environment.Variable("flink_recovery_mode", "zookeeper"),
                new Mesos.Environment.Variable("flink_recovery_zookeeper_quorum", app.get("zkUrl")),
                new Mesos.Environment.Variable("flink_recovery_zookeeper_storageDir", "/data/zk")
            ]), // Environment
            false, // Is shell?
            null, // Command
            ["jobmanager"], // Arguments
            null // User
        ),
        "resources": {
            "cpus": 0.5,
            "mem": 256,
            "ports": 2,
            "disk": 0
        },
        "healthChecks": null, // Add your health checks here
        "labels": null // Add your labels (an array of { "key": "value" } objects)
    },
    "taskmanagers": {
        "priority": 2,
        "instances": 2,
        "allowScaling": true,
        "executorInfo": null, // Can take a Mesos.ExecutorInfo object
        "containerInfo": ContainerInfo, // Mesos.ContainerInfo object
        "commandInfo": new Mesos.CommandInfo( // Strangely, this is needed, even when specifying ContainerInfo...
            null, // URI
            new Mesos.Environment([
                new Mesos.Environment.Variable("flink_recovery_mode", "zookeeper"),
                new Mesos.Environment.Variable("flink_recovery_zookeeper_quorum", app.get("zkUrl")),
                new Mesos.Environment.Variable("flink_recovery_zookeeper_storageDir", "/data/zk"),
                new Mesos.Environment.Variable("flink_taskmanager_tmp_dirs", "/data/tasks"),
                new Mesos.Environment.Variable("flink_blob_storage_directory", "/data/blobs"),
                new Mesos.Environment.Variable("flink_state_backend", "filesystem"),
                new Mesos.Environment.Variable("flink_taskmanager_numberOfTaskSlots", "1"),
                new Mesos.Environment.Variable("flink_taskmanager_heap_mb", "1536")
            ]), // Environment
            false, // Is shell?
            null, // Command
            ["taskmanager"], // Arguments
            null // User
        ),
        "resources": {
            "cpus": 0.5,
            "mem": 1536,
            "ports": 3,
            "disk": 0
        },
        "healthChecks": null, // Add your health checks here
        "labels": null // Add your labels (an array of { "key": "value" } objects)
    }
};
```

### The frameworkConfiguration object

The `frameworkConfiguration` object defined the basic properties and abilities of the framework. Please specifically have a look at the [scheduler docs](https://github.com/tobilg/mesos-framework/blob/master/README.md#scheduler) to get a complete overview of all usable properties.
 
As [mesos-framework](https://github.com/tobilg/mesos-framework) is a wrapper around the Master's Scheduler and Executor HTTP APIs, a `masterUrl` needs to be specified. If you use Mesos DNS in your cluster, you don't need to set anything here, because `leader.mesos` will be automatically used to discover the leading Mesos Master.

You should definitely customize the `frameworkName` though, keep in mind that the framework name is the identifier used for ZK persistency!

```javascript
// The framework's overall configuration
var frameworkConfiguration = {
    "masterUrl": process.env.MASTER_IP || "leader.mesos",
    "port": 5050,
    "frameworkName": "Mesos-Framework-Boilerplate",
    "logging": {
        "path": path.join(__dirname , "/logs"),
        "fileName": "mesos-framework-boilerplate.log",
        "level": app.get("logLevel")
    },
    "tasks": frameworkTasks
};
```

## Dockerfile

The Dockerfile uses a minimal Alpine Linux/Node.js 6.3 base image.  

```bash
FROM mhart/alpine-node:6.3.0

MAINTAINER tobilg@gmail.com

# Setup of the prerequisites
RUN apk add --no-cache git && \
    apk add --no-cache ca-certificates openssl && \
    mkdir -p /mnt/mesos/sandbox/logs && \
    npm set progress=false

# Set application name
ENV APP_NAME mesos-framework-boilerplate

# Set application directory
ENV APP_DIR /usr/local/${APP_NAME}

# Set node env to production, so that npm install doesn't install the devDependencies
ENV NODE_ENV production

# Add application
ADD . ${APP_DIR}

# Change the workdir to the app's directory
WORKDIR ${APP_DIR}

# Setup of the application
RUN npm install --silent && \
    npm install bower -g && \
    bower install --allow-root

CMD ["sh", "./get_creds.sh"]
```

### Customization

You should edit the `APP_NAME` variable, as well as the `MAINTAINER`.

### Building your image

You should create a [Docker Hub](https://hub.docker.com/) account if you aren't using a private Docker registry, and [configure](https://docs.docker.com/engine/tutorials/dockerrepos/) your Docker local installation accordingly.
  
Build your Docker image from the project's root path like this:

```bash
docker build -t USERNAME/IMAGENAME:TAG .
```

where `USERNAME` is your Docker Hub username, and `IMAGENAME` is the desired name of the image. If you just want to create a `latest` image, you can leave out the `TAG`.

### Pushing to the Docker Hub registry

You can push the image like this:

```bash
docker push USERNAME/IMAGENAME:TAG
```

## Running on Mesos with Marathon

Once your Docker image has been built and pushed to a registry, you can use it to deploy your framework on Mesos via Marathon.

### Basic application definition

```javascript
{
  "id": "mesos-framework-boilerplate",
  "container": {
    "docker": {
      "image": "tobilg/mesos-framework-boilerplate:latest",
      "network": "HOST",
      "forcePullImage": true
    },
    "type": "DOCKER"
  },
  "cpus": 0.5,
  "mem": 256,
  "instances": 1,
  "healthChecks": [
    {
      "path": "/health",
      "protocol": "HTTP",
      "gracePeriodSeconds": 30,
      "intervalSeconds": 10,
      "timeoutSeconds": 20,
      "maxConsecutiveFailures": 3,
      "ignoreHttp1xx": false,
      "portIndex": 0
    }
  ],
  "labels": {
    "DCOS_SERVICE_SCHEME": "http",
    "DCOS_SERVICE_NAME": "<FRAMEWORK_NAME>",
    "DCOS_PACKAGE_FRAMEWORK_NAME": "<FRAMEWORK_NAME>",
    "DCOS_SERVICE_PORT_INDEX": "0"
   },
  "ports": [0],
  "env": {
    "LOG_LEVEL": "info",
    "TASK_DEF_NUM": "1",
    "TASK0_ENV": "{}",
    "TASK0_IMAGE": "alpine",
    "TASK0_NUM_INSTANCES": "3",
    "TASK0_URI": "<OPTIONAL_URI>",
    "FRAMEWORK_NAME": "<FRAMEWORK_NAME>",
    "TASK0_NAME": "alpine",
    "TASK0_CPUS": "0.5",
    "TASK0_MEM": "512",
    "TASK0_HEALTHCHECK": "<HEALTHCHECK_URL>",
    "TASK0_HEALTHCHECK_PORT": "1",
    "TASK0_FIXED_PORTS": "<PORTS>",
    "TASK0_PORT_NUM": "1",
    "TASK0_CONTAINER_PARAMS": "[]",
    "TASK0_ARGS": "[\"sleep\", \"100000\"]",
    "TASK0_NOCOLOCATION": "false",
    "TASK0_CONTAINER_PRIVILEGED": "false",
    "FRAMEWORK_NAME_BACKGROUND": "#ecfdf0"
  }
}
```

### Customization

You should replace `FRAMEWORK_NAME` with a meaningful name for the Marathon app. Furthermore, it is necessary to insert the correct Docker image name instead of `REGISTRY_URL/USERNAME/IMAGENAME:TAG`.

Furthermore, you can customize the `cpus` and `mem` settings, although they should suffice as already configured. Keep in mind that the `mesos-framework` module doesn't yet support scheduler failover (HA), so running more than one instance has no benefit, it will clutter the logs with failovers and reconnections. This will be changed in future releases.  

You can customize the tasks environment using the TASK# environment variable (gets a JSON object as string), container params, ports, args, and all other environment variables.  
The required ones are: TASK_DEF_NUM, FRAMEWORK_NAME, TASK#_NAME, TASK#_NUM_INSTANCES, TASK#_CPUS, TASK#_MEM, TASK#_IMAGE.

If you want to support authentication, you need to add an environment variable named CREDENTIALS_URL with a URL to a shell file that sets the authentication related variables ("GITLAB_APP_ID", "GITLAB_APP_SECRET", "GITLAB_CALLBACK_URL" - for GitLab and/or "GOOGLE_CLIENT_ID", "GOOGLE_CLIENT_SECRET", "GOOGLE_CALLBACK_URL" - for Google, required: AUTH_COOKIE_ENCRYPTION_KEY, optionally: GITLAB_URL, GOOGLE_FILTER, GOOGLE_SCOPE), the callback URLs are /auth/gitlab/callback and /auth/google/callback, naturally you should add the host and port of your mesos agent/load balancer.

### Launching

Once you prepared your `marathon.json`, you can launch it from the command-line in the root folder of the project like this (replace `MARATHON_URL` with a valid Marathon URL):

```bash
curl -H "Accept: application/json" -H "Content-Type:application/json" -XPUT "MARATHON_URL" -d @marathon.json
```

You should receive a `201` status, and see the application in the Marathon UI accordingly (Go to the "Frameworks" tab, and search for the framework's name under "Active Frameworks"). Click on the link (in the "Host" column), this should lead you to the frameworks UI in a new browser tab.

## UI screenshots

### Dashboard

![Dashboard](docs/screenshots/dashboard.png)

### Scaling

![Scaling](docs/screenshots/scaling.png)

### Tasks

![Tasks](docs/screenshots/tasks.png)

### Configuration

![Configuration](docs/screenshots/configuration.png)

### Logs

![Logs](docs/screenshots/logs.png)
