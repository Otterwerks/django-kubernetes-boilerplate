# Django Postgres

Use the manifests and information in this branch to deploy your Django app and Postgres on Kubernetes.


## Configuration

1. Modify your Django `settings.py` file so that the DATABASES section looks like this:

<pre>
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'applicationtrackerdb',
        'USER': os.getenv('POSTGRES_USER'),
        'PASSWORD': os.getenv('POSTGRES_PASSWORD'),
        'HOST': os.getenv('POSTGRES_HOST'),
        'PORT': os.getenv('POSTGRES_PORT', 5432)
    }
}
</pre>

2. Make sure your `requirements.txt` is in the Django project root directory (where `manage.py` is). If you don't have this file, generate it by entering into your Django project's Python environment and using the command `pip freeze > requirements.txt`.

3. Configure the status check url path /alive in your Django app by:
- adding `path('alive', views.alive, name='alive')`, to `urls.py`
- in `views.py`, adding `from django.http import HttpResponse` and 
<pre>
def alive(request):
  return HttpResponse(status=200)
</pre>


4. The `Dockerfile` in this directory is currently configured with `FROM arm32v7/python:3.6`, change this to `FROM python:3.6-buster` if you are not deploying on an ARM based Kubernetes cluster. 

5. Place `Dockerfile` into the Django project root directory, build and push the image to DockerHub. 

6. Open `manifests/django-deployment.yaml` and use your Docker image's tag to replace the placeholder for both the deployment image as well as the init container image.

7. Postgres needs a volume to store data, the manifests are currently designed to use dynamic provisioning of persistent volumes using a default Kubernetes StorageClass. If your Kubernetes does not have dynamic provisioning configured you may modify the `postgres-deployment.yaml` to use another form of storage such as local, hostPath, or even emptyDir (for testing). See [https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes) for more information and examples.

8. Make sure you are using the correct Postgres image in `postgres-deployment.yaml`, remove `arm32v7` if you are not deploying to an ARM based Kubernetes cluster.

9. Set up the Django service in `django-service.yaml`. This will be specific to your environment and how you want to access your app. I use an ingress to forward requests with a specific hostname to this service but you may want to begin with [a different option](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0).



## Deployment

With your manifests configured and your image push to DockerHub, follow these steps to deploy:

 Create a namespace, 'django-postgres' by default, `kubectly create namespace django-postgres`, then apply manifests using `kubectl apply -f <manifestname.yaml>` in this order:

 1. `postgres-secret.yaml`, `postgres-service.yaml`, `postgres-pvc.yaml` (if applicable)
 2. `postgres-deployment.yaml`
 3. once`kubectl -n django-postgres get pods` shows `1/1 Ready` for the Postgres pod, apply `django-service.yaml` and `django-deployment.yaml` 

## Clean Up
`kubectl delete namespace django-postgres` will remove all resources created from these manifests. 

## Troubleshooting

If you are unable to access your app, here are some things they may have gone wrong.

### Are the pods ready?
The deployment uses a Kubernetes HTTP Readiness Probe to set the status of the Django app pods, if `http://<hostname or ip>/alive` isn't reachable then your app pods will never be set to the `Ready` state to receive traffic. Try running the app image with Docker and test if you are able to receive a response at the `/alive` path. Alternatively you can remove both the Readiness and Liveness probes from `django-deployment.yaml`.

### Did the Django image build correctly?
This repository is configured by default to work on ARM. Double check that the Django app image didn't build from the ARM32v7 Python image if you are not deploying on ARM, also check that the directory and file structure is correct, `Dockerfile` and `requirements.txt` should be placed in the same directory as `manage.py`.






