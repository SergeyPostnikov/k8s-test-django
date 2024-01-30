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

## How to run in a cluster
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
- Create a django-config.yml file in the kubernetes directory with the following manifest:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
data:
  DATABASE_URL: postgres://USER:PASSWORD@HOST:PORT/NAME
  SECRET_KEY: 123456
  DEBUG: 'false'
  ALLOWED_HOSTS: star-burger.test
```
- Apply the created ConfigMap: 
```shell
kubectl apply -f .\kubernetes\django-config.yml
```
- Apply all manifests from the kubernetes folder: 
```
kubectl apply -f .\kubernetes\
```

## Environment Variables

The Django image reads settings from environment variables:

`SECRET_KEY` -- Django's mandatory secret setting. It is the salt for generating hashes. The value can be anything, it's essential that it is not known to anyone. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- Django's setting to enable the debug mode. It takes values TRUE or FALSE. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- Django's setting with a list of allowed addresses. If a request comes to another address, the site will respond with a 400 error. You can list multiple addresses separated by commas, for example, 127.0.0.1,192.168.0.1,site.test. [Django Documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- Address to connect to the PostgreSQL database. The site does not support other databases. [Format of the record](https://github.com/jacobian/dj-database-url#url-schema).
