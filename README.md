# Django Site

Dockerized Django site for experiments with Kubernetes.

Inside the Django container, it runs with Nginx Unit (not to be confused with Nginx). The Nginx Unit server serves static and media files as a web server and acts as an application server, running Python and Django. Thus, Nginx Unit replaces the combination of two services: Nginx and Gunicorn/uWSGI. [Learn more about Nginx Unit](https://unit.nginx.org/).

## How to run the local version

Run the database and the site:

```shell
$ docker compose up
```

In a new terminal, without turning off the site, run the commands to set up the database:


```shell
$ docker compose run --rm web ./manage.py migrate  
$ docker compose run --rm web ./manage.py createsuperuser  
```

For fine-tuning Docker Compose, use environment variables. Their names differ from those set by the docker image to avoid naming conflicts. Several images are configured within docker-compose.yaml, each with its own environment variables, so their names may accidentally overlap. To avoid conflicts, prefixes by the service name are added to the environment variable names. You can find the list of available variables inside the docker-compose.yml file.

## How to Run in Yandex Cloud

To make the website accessible to users on the internet, you'll need a cluster with a public IP address and domain. You can rent such a cluster from cloud hosting providers and configure it accordingly.

In this project, a YandexCloud cluster is prepared and configured in advance:

### Cluster Resources:
* A domain `edu-furious-nobel.sirius-k8s.dvmn.org` is allocated. Requests are handled by the Yandex Application Load Balancer.
* A database is created in Yandex Managed Service for PostgreSQL. Access credentials are securely stored in K8s secrets.
* A router is created in the Yandex Application Load Balancer, distributing incoming network requests to different NodePort instances of the K8s cluster.
* S3 Bucket is configured, with tokens and other access settings for Object Storage API securely stored in K8s secrets.

### Placing the Image in Docker Registry
To deploy the application, you need to place its image in [Docker Hub](https://hub.docker.com):
* Build the image locally:
```shell
docker build -t image_name:tag -f path_to_Dockerfile .
```
* Log in to Docker Hub:
```shell
docker login
```
* Tag the local image:
```shell
docker tag image_name:tag your_username/repository_name:tag
```
* Upload the image to Docker Hub:
```shell
docker push your_username/repository_name:tag
```

### Obtaining an SSL Certificate to Connect to the PostgreSQL Database
PostgreSQL hosts with public access support only encrypted connections. Obtain an SSL certificate to use them.

* For Linux(Bash)/macOS(Zsh):
```shell
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```
* For Windows(PowerShell):
```shell
mkdir $HOME\.postgresql; curl.exe -o $HOME\.postgresql\root.crt https://storage.yandexcloud.net/cloud-certs/CA.pem
```

The certificate will be saved in the file `$HOME\.postgresql\root.crt`.

### Deploying the Application in the Cluster

1. Update `k8s_prod/django-config.yaml` with your DEBUG and ALLOWED_HOSTS settings.

2. Apply the configuration file (example `django-config-example.yaml`):
```bash
kubectl apply -f k8s_prod/django-config.yaml  
```

3. Create your `k8s_prod/django-secret.yaml` with the following command:
```bash
echo your_secret | base64
```
Then, place the resulting secret in the manifest in the appropriate places.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  SECRET_KEY: your_key
  DATABASE_URL: your_cloud_db_url
```

4. Apply the secret:
```shell
kubectl apply -f k8s_prod/django-secret.yaml  
```
Alternatively, you can create the secret using the following command in your terminal:
```bash
kubectl create secret generic django-secrets \
     --from-literal=SECRET_KEY=your_key \
     --from-literal=DATABASE_URL=postgres://USER:PASSWORD@HOST:PORT/NAME \
     -n your_namespace
```

5. Start the deployment:
```bash
kubectl apply -f k8s_prod/deployment-django.yaml
```

6. Create the service:
```bash
kubectl apply -f k8s_prod/django-service.yaml
```

7. Apply migrations to the database and copy static files to the bucket:
```bash
kubectl apply -f k8s_prod/migrate-job.yaml
```

8. To clear sessions, run django-clearsessions:
```bash
kubectl apply -f k8s_prod/clearsessions-cron-job.yaml
```

The website is be accessible at [edu-furious-nobel.sirius-k8s.dvmn.org](https://edu-furious-nobel.sirius-k8s.dvmn.org/)

## How to run in a minikube cluster
- before start you could add local alias to minikube ip
```bash
echo "$(minikube ip) your.domain.test" | sudo tee -a /etc/hosts
```
- The file .\kubernetes\deployment-django.yaml specifies the image from which the pods will be launched: image: django_app.
To create the image, use:
```shell
cd backend_main_django
eval $(minikube docker-env)
docker build -t django_app .
```
- Create a Postgres pod using Helm: 
```shell
helm install django-db oci://registry-1.docker.io/bitnamicharts/postgresql`
```
- [Create a database](https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e)

### Creating a Secret for Django

For secure storage of confidential data in Kubernetes, we will be using secrets. Below are instructions for creating a secret for Django.
1. Open a terminal and execute the following command to create the secret:

```bash
   kubectl create secret generic django-secrets \
     --from-literal=SECRET_KEY=your_key \
     --from-literal=DATABASE_URL=postgres://USER:PASSWORD@HOST:PORT/NAME \
```
2. Ensure that the secret has been successfully created by running:

   ```bash
   kubectl get secret django-secrets
   ```

   You should see information about the created secret, including its type (Opaque) and data.

3. Now you can use this secret in your Kubernetes manifests (e.g., in deployments and config maps) to securely store confidential information.

```
## Creating a configMap for Django
- Create a django-config.yml file in the kubernetes directory with the following manifest:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
data:
  DEBUG: 'False'
  ALLOWED_HOSTS: star-burger.test
```


Replace the values of `SECRET_KEY`, `DATABASE_URL`, `ALLOWED_HOSTS`, and `DEBUG` with the actual values you use in your Django application.

- Apply all manifests from the kubernetes folder: 
```
kubectl apply -f kubernetes\
```

## Environment Variables

The Django image reads settings from environment variables:

`SECRET_KEY` -- Django's mandatory secret setting. It is the salt for generating hashes. The value can be anything, it's essential that it is not known to anyone. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- Django's setting to enable the debug mode. It takes values TRUE or FALSE. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- Django's setting with a list of allowed addresses. If a request comes to another address, the site will respond with a 400 error. You can list multiple addresses separated by commas, for example, 127.0.0.1,192.168.0.1,site.test. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- Address to connect to the PostgreSQL database. The site does not support other databases. [Format of the record](https://github.com/jacobian/dj-database-url#url-schema).
