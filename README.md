# telemetry-airflow
Airflow is a platform to programmatically author, schedule and monitor workflows.

When workflows are defined as code, they become more maintainable, versionable, testable, and collaborative.

Use Airflow to author workflows as directed acyclic graphs (DAGs) of tasks. The Airflow scheduler executes your tasks on an array of workers while following the specified dependencies. Rich command line utilities make performing complex surgeries on DAGs a snap. The rich user interface makes it easy to visualize pipelines running in production, monitor progress, and troubleshoot issues when needed.

### Build Container

An Airflow container can be built with 

```bash
docker build -t mozdata/telemetry-airflow .
```

and pushed to Docker hub with
```bash
docker push mozdata/telemetry-airflow
```

### Testing

When developing and testing a new DAG, it should be configured to write data to the `telemetry-test-bucket` bucket, and **not** the `airflow_bucket` specified in the ansible environment. Only when the new code and data have been validated should it begin writing to the `airflow_bucket`.

A single task, e.g. `spark`, of an Airflow dag, e.g. `example`, can be run with an execution date, e.g. `2016-01-01`, in the `dev` environment with:
```bash
AWS_SECRET_ACCESS_KEY=... AWS_ACCESS_KEY_ID=... \
ansible-playbook ansible/deploy_local.yml -e '@ansible/envs/dev.yml' -e "command='test example spark 20160101'"
```

The container will run the desired task to completion (or failure). Note that if the container is stopped during the execution of a task, the task will
be aborted. In the example's case, the Spark job will be terminated. 

The logs of the task can be inspected in real-time with:
```bash
docker logs -f files_scheduler_1
```

### Local Deployment

Assuming you are on OS X, first create a docker machine with a sufficient amount of memory with e.g.:
```bash
docker-machine create -d virtualbox --virtualbox-memory 4096 default
```

To deploy the Airflow container on the docker engine, with its required dependencies, run:
```bash
ansible-playbook ansible/deploy_local.yml -e '@ansible/envs/dev.yml'
echo "Airflow web console should now be running locally at http://$(docker-machine ip default):8080"
```

Note that this will start running all the DAGs with a start date in the past! To avoid that do not pass the AWS credentials.

If you get a message saying "Couldn't connect to Docker daemon - you might need to run `docker-machine start default`.", try the following:
```bash
docker-machine start default
eval "$(docker-machine env default)"
```

You can now connect to your local Airflow web console with a URL like `http://192.168.99.100:8080` (see above for how to identify the exact IP address).

### Remote Deployment

In order to deploy Airflow to e.g. the `stage` environment, an ECS cluster has to be created first with at least one container instance:
```bash
ansible-playbook ansible/provision_aws.yml -e '@ansible/envs/stage.yml'
```

Once the ECS cluster is up and running, Airflow can be (re)deployed with:
```bash
ansible-playbook ansible/deploy_aws.yml -e '@ansible/envs/stage.yml'
```

### Debugging

If you get an error about `ecs-cli` missing, follow [these steps](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html) to install it.

Some useful docker tricks for development and debugging:

```bash
# Stop all docker containers:
docker stop $(docker ps -aq)

# Remove any leftover docker volumes:
docker volume rm $(docker volume ls -qf dangling=true)
```
