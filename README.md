# django to kubernetes deployment example

This is an example of deployment django application to kubernetes cluster in google container engine with CircleCI.

### Goals

* Automate building of local environment 
* Deploy application to sandbox and production environment
* Run the test on a CI server after the code is committed via git

### Why kubernetes?

Kubernetes is an open-source platform for automating deployment, scaling, and operations of application containers across clusters of hosts, providing container-centric infrastructure.
With Kubernetes, you are able to quickly and efficiently respond to customer demand:

* Deploy your applications quickly and predictably.
* Scale your applications on the fly.
* Seamlessly roll out new features.
* Optimize use of your hardware by using only the resources you need.

Many cloud providers have kubernetes as a servive (google, aws etc.), so it's easy to administrate and maintain.

### Requirements

* CircleCI
* google container engine (kubernetes)
* gcloud and kubectl tools
* docker and docker-compose

### Local environment

You can launch django polls locally first. Clone repository and run in the repository directory:
```bash
docker-compose build
docker-compose up -d
docker exec -it djangopollskubernetes_django_1 python manage.py migrate
docker exec -it djangopollskubernetes_django_1 python manage.py createsuperuser
```

Django polls app is started now, but a database is still empty. The next step is to complete DB migration:
```bash
docker exec -it djangopollskubernetes_django_1 python manage.py migrate
```
DB is initialized. Let's create django user:
```bash
docker exec -it djangopollskubernetes_django_1 python manage.py createsuperuser
```
Django polls app is available now by http://localhost:8000

Basic urls:

 * http://localhost:8000/polls
 * http://localhost:8000/admin

Unit tests can be executed with the command below:
```bash
docker exec -it djangopollskubernetes_django_1 python manage.py test
```

### Google container engine

We will deploy polls to GKE. It offers two services we need:

* kubernetes
* container registry

Create a basic cluster in google cloud engine. More info you can find in [GKE documentation](https://cloud.google.com/container-engine/docs/clusters/operations). Also write zone and project ID to some place.

Obtain gcloud key in Cloud Platform console. Need to create [Service account](https://developers.google.com/api-client-library/php/auth/service-accounts) first to do it. Please note that gcloud key should be in json fromat.

Install [gcloud tools](https://cloud.google.com/sdk/gcloud/) and [kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/). Let's check kubectl now:
```bash
gcloud auth activate-service-account --key-file=key.json
gcloud container clusters get-credentials cluster-1 --zone <GKE zone> --project <GKE project ID>
kubectl proxy
```

Open http://127.0.0.1:8001/ui in browser, you should see kubernetes cluster web UI. Basic kubernetes cluster setup has finished. Now it's time to create DB instances.

### Database server install

You can use any mysql server (for example Amazon RDS or gcloud SQL), but in this example we will consider mysql installation to kubernetes cluster. You can do by it by running the commands below:
```bash
gcloud config set project <GKE project ID>
gcloud compute disks create --size=20GB mysql-disk
kubectl create -f kubernetes/mysql-volume.yaml 
kubectl create -f kubernetes/mysql-deployment.yaml
kubectl run -it --rm --image=mysql:5.6 mysql-cli -- mysql -h mysql -ppassword -e "CREATE DATABASE django_polls;"
```
We've just created production mysql instance. It's accessiable from kuberenetes only and has 20GB persistant volume. You can do the same for sandbox, but need to change GKE project.

### CircleCI

We will use [CircleCI](http://circleci.com) for continuous integration and deployment. Connect repository to CircleCI. More info about it in its [documentation](https://circleci.com/docs/1.0/getting-started/). Add production environment variables in CircleCI:

* PROD_GCLOUD_KEY - json key to access gcloud services
* PROD_GKE_ZONE - project zone
* PROD_GKE_PROJECT - project ID
* PROD_GKE_CLUSTER - kubernetes clluser ID
* PROD_MYSQL_DB - name of DB
* PROD_MYSQL_USER - mysql user
* PROD_MYSQL_PASSWORD - mysql password
* PROD_MYSQL_SERVER - address of mysql server
* PROD_MYSQL_PORT - mysql server port
* PROD_REPLICAS_NUM - number of running instances of the application

And sandbox environment variables:

* SANDBOX_GCLOUD_KEY - json key to access gcloud services
* SANDBOX_GKE_ZONE - project zone
* SANDBOX_GKE_PROJECT - project ID
* SANDBOX_GKE_CLUSTER - kubernetes clluser ID
* SANDBOX_MYSQL_DB - name of DB
* SANDBOX_MYSQL_USER - mysql user
* SANDBOX_MYSQL_PASSWORD - mysql password
* SANDBOX_MYSQL_SERVER - address of mysql server
* SANDBOX_MYSQL_PORT - mysql server port
* SANDBOX_REPLICAS_NUM - number of running instances of the application


Build and deployment proccess is described in .circle.yml file in the repository. You can run build for production and sandbox branches now. The following steps will be done:
* Build docker image with django and push it to docker registry
* Run unit tests
* Run DB migrations
* Deploy application to kubernetes

To expose production application and configure load balancing run:
```bash
kubectl create -f kubernetes/lb-production.yaml
```
for sandbox:
```bash
kubectl create -f kubernetes/lb-sandbox.yaml
```

IP address to connect you can find in kubernetes UI or through kubectl:
```bash
kubectl describe services djangopolls-production | grep Ingress
```
