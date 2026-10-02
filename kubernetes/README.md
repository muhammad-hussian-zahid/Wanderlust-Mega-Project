# Wanderlust Deployment on Kubernetes (kind) using GitHub Codespaces

### In this project, we will learn how to deploy the Wanderlust MERN stack application (Frontend, Backend, MongoDB, Redis) on a Kubernetes cluster created with **kind**, running inside a **GitHub Codespace**. No cloud account (AWS, etc.) is required.

### Pre-requisites to implement this project:
- A GitHub account.
- A Docker Hub account (to store the frontend and backend images).
- A Codespace with **4 cores / 16 GB RAM** (a free account gives about 30 hours per month on this machine size, so stop the Codespace when you are not using it).

### Tech stack used in this project:
- GitHub Codespaces (Cloud dev machine)
- Docker and Docker Hub (Containerization and image registry)
- kind (Kubernetes in Docker)
- kubectl (Kubernetes CLI)
- MongoDB (Database)
- Redis (Caching)

### Docker images used in this project:

| Component | Image |
|---|---|
| Frontend | `hussain968/frontend-wanderlust:latest` |
| Backend | `hussain968/backend-wanderlust:latest` |
| MongoDB | `mongo:latest` |
| Redis | `redis:latest` |

> Note: `hussain968` is the Docker Hub username used in this guide. If you are following this guide with your own Docker Hub account, replace `hussain968` with your username in every command below.

### How the application is exposed:

| Component | Container port | NodePort | Codespace URL |
|---|---|---|---|
| Frontend | 5173 | 31000 | `https://<codespace-name>-31000.app.github.dev` |
| Backend | 8080 | 31100 | `https://<codespace-name>-31100.app.github.dev` |
| MongoDB | 27017 | internal only | - |
| Redis | 6379 | internal only | - |

#
## Steps for Kubernetes deployment:

1) Fork the repository :

- Open `https://github.com/DevMadhup/Wanderlust-Mega-Project`
- Click **Fork** (top right) and then **Create fork**

#
2) Create the Codespace :

- Open your fork, click the green **Code** button and open the **Codespaces** tab
- Click **...** and then **New with options**
- Select the **4-core** machine type and click **Create codespace**
- Wait until VS Code opens in your browser, then open the terminal (`` Ctrl + ` ``)

#
3) Verify Docker and kubectl are installed :
```bash
docker --version
kubectl version --client
```

#
4) Install kind :
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

#
5) Navigate to the project directory :
```bash
cd /workspaces/Wanderlust-Mega-Project
```

#
6) Set your Codespace URLs as variables :

Every forwarded port in a Codespace gets its own URL. These variables are used in the next steps.

```bash
FRONT="https://${CODESPACE_NAME}-31000.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}"
BACK="https://${CODESPACE_NAME}-31100.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}"

echo $FRONT
echo $BACK
```

> Note: These variables only exist in the current terminal. If you open a new terminal, run the commands above again.

#
7) Update the frontend environment file :

The frontend needs the backend URL (`VITE_API_PATH`).

```bash
sed -i "s|^VITE_API_PATH=.*|VITE_API_PATH=\"$BACK\"|" frontend/.env.docker
cat frontend/.env.docker
```

#
8) Update the backend environment file :

The backend needs the frontend URL (`FRONTEND_URL`) so that it allows requests from your frontend.

```bash
sed -i "s|^FRONTEND_URL=.*|FRONTEND_URL=\"$FRONT\"|" backend/.env.docker
cat backend/.env.docker
```

Make sure these two values are unchanged, because they match the Kubernetes service names:
- `MONGODB_URI="mongodb://mongo-service/wanderlust"`
- `REDIS_URL="redis://redis-service:6379"`

**(Recommended)** Generate your own `JWT_SECRET` instead of using the one in the repository:
```bash
sed -i "s|^JWT_SECRET=.*|JWT_SECRET=$(openssl rand -hex 64)|" backend/.env.docker
```

> Note: Do not commit the `.env.docker` files to GitHub, because they contain your Codespace URLs and secret.

#
9) Build the frontend and backend Docker images :

The URLs from steps 7 and 8 are copied into the images during the build, so always build **after** editing the env files.

```bash
docker build -t hussain968/frontend-wanderlust:latest ./frontend
docker build -t hussain968/backend-wanderlust:latest ./backend
```

Check the images :
```bash
docker images | grep wanderlust
```

#
10) Login to Docker Hub and push the images :

```bash
docker login
```

```bash
docker push hussain968/frontend-wanderlust:latest
docker push hussain968/backend-wanderlust:latest
```

> Note: The images contain your Codespace URLs and `JWT_SECRET`. Consider making the two Docker Hub repositories **private** (Docker Hub > repository > Settings).

#
11) Pull the MongoDB and Redis images :
```bash
docker pull --platform linux/amd64 mongo:latest
docker pull --platform linux/amd64 redis:latest
```

#
12) Create the kind cluster :

The config file maps NodePorts `31000` and `31100` so that your application can be reached from outside the cluster.

```bash
cat <<EOT > kubernetes/kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31000
    hostPort: 31000
  - containerPort: 31100
    hostPort: 31100
EOT
```

```bash
kind create cluster --name wanderlust --config kubernetes/kind-config.yaml
```

Verify the node is ready :
```bash
kubectl get nodes
```

#
13) Load all four images into the kind cluster :

The cluster runs inside a Docker container, so the images must be copied into it. This may take a few minutes (MongoDB is the largest image), so wait until each command finishes.

```bash
for IMG in hussain968/frontend-wanderlust:latest hussain968/backend-wanderlust:latest mongo:latest redis:latest; do
  docker save --platform linux/amd64 $IMG | docker exec -i wanderlust-control-plane ctr -n k8s.io images import --digests -
done
```

Verify that all four images are present in the cluster :
```bash
docker exec wanderlust-control-plane crictl images | grep -E "hussain968|mongo|redis"
```

You should see `frontend-wanderlust`, `backend-wanderlust`, `mongo` and `redis`.

#
14) Navigate to the kubernetes directory :
```bash
cd kubernetes
```

#
15) Update the manifests to use your Docker Hub images :

This points the frontend and backend manifests to your images and tells Kubernetes to use images that are already present in the cluster instead of downloading them again.

```bash
sed -i 's|image: .*\(wanderlust-frontend\|frontend-wanderlust\).*|image: hussain968/frontend-wanderlust:latest\n          imagePullPolicy: IfNotPresent|' frontend.yaml
sed -i 's|image: .*\(wanderlust-backend\|backend-wanderlust\).*|image: hussain968/backend-wanderlust:latest\n          imagePullPolicy: IfNotPresent|' backend.yaml
sed -i 's|image: mongo$|image: mongo\n          imagePullPolicy: IfNotPresent|' mongodb.yaml
sed -i 's|image: redis$|image: redis\n          imagePullPolicy: IfNotPresent|' redis.yaml
```

Verify the changes :
```bash
grep -A1 "image:" *.yaml
```

Each `image:` line should be followed by `imagePullPolicy: IfNotPresent`.

> Note: Run the `sed` commands only once, otherwise the `imagePullPolicy` line will be duplicated.

#
16) Create the Kubernetes namespace :
```bash
kubectl create namespace wanderlust
```

#
17) Update the Kubernetes config context :

This sets `wanderlust` as the default namespace, so you do not need to add `-n wanderlust` to every command.

```bash
kubectl config set-context --current --namespace wanderlust
```

#
18) Apply the manifest files in the below order :

> Note: Apply the files one by one by name. Do not run `kubectl apply -f .` because this folder also contains `kind-config.yaml`, which is a kind file and not a Kubernetes manifest.

- Create persistent volume and persistent volume claim :
```bash
kubectl apply -f persistentVolume.yaml
kubectl apply -f persistentVolumeClaim.yaml
```

- Create MongoDB deployment and service :
```bash
kubectl apply -f mongodb.yaml
```

- Create Redis deployment and service :
```bash
kubectl apply -f redis.yaml
```

- Wait until MongoDB and Redis are ready, otherwise the backend will not be able to connect :
```bash
kubectl wait --for=condition=ready pod -l app=mongo --timeout=180s
kubectl wait --for=condition=ready pod -l app=redis --timeout=180s
```

- Create Backend deployment and service :
```bash
kubectl apply -f backend.yaml
```

- Create Frontend deployment and service :
```bash
kubectl apply -f frontend.yaml
```

#
19) Check all deployments and services :
```bash
kubectl get all
kubectl get pv,pvc
```

All four pods should be in `Running` state with `1/1` ready, and the PV and PVC should be `Bound`. Pods may take a minute or two to start.

```bash
kubectl get pods -w
```
(Press `Ctrl + C` to stop watching.)

#
20) Check the logs of the backend :

> Note: This confirms the backend is connected to MongoDB and Redis. If the backend started before the databases were ready, recreate its pod with `kubectl delete pod -l app=backend`.

```bash
kubectl logs -l app=backend --tail=20
```

#
21) Make the application ports public :

Codespaces ports are private by default. The backend must be public because your browser calls it directly.

```bash
gh codespace ports visibility 31000:public 31100:public -c $CODESPACE_NAME
```

Or use the **Ports** tab in VS Code: right-click each port, choose **Port Visibility** and then **Public**. If the ports are not listed, click **Add Port** and add `31000` and `31100`.

#
22) Verify the backend :

```bash
curl -i http://localhost:31100/
curl -i "https://${CODESPACE_NAME}-31100.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}/"
```

Any HTTP response means the backend is running and reachable.

#
23) Access your application in Chrome :

Frontend :
```bash
echo "https://${CODESPACE_NAME}-31000.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}"
```

Backend :
```bash
echo "https://${CODESPACE_NAME}-31100.${GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN}"
```

Open the URLs in your browser. If Codespaces shows a warning page for the development port, click **Continue**. Your Wanderlust application is now deployed on Kubernetes.

#
## Stop and restart the Codespace

**Stop the Codespace** (saves your free hours; your cluster, images and data are kept) :

- Browser: open `https://github.com/codespaces`, click **...** next to your Codespace and choose **Stop codespace**
- Terminal:
```bash
gh codespace stop -c $CODESPACE_NAME
```

> Note: Closing the browser tab does not stop the Codespace. It keeps running until the inactivity timeout (30 minutes by default).

**Start it again later :**

1) Open `https://github.com/codespaces` and click your Codespace.

2) Start the kind cluster container :
```bash
docker start wanderlust-control-plane
```

3) If `kubectl` cannot connect, refresh the connection :
```bash
kind export kubeconfig --name wanderlust
```

4) Wait a minute and check that all pods are running again (no need to redeploy) :
```bash
kubectl get nodes
kubectl get pods
```

5) If a pod stays in an error state, recreate it (use `mongo`, `redis` or `frontend` for the others) :
```bash
kubectl delete pod -l app=backend
```

6) Make the ports public again :
```bash
gh codespace ports visibility 31000:public 31100:public -c $CODESPACE_NAME
```

#
## Clean up

Delete only the cluster (keeps the Codespace) :
```bash
kind delete cluster --name wanderlust
```

Delete the Codespace when you have finished the project (this removes everything, including the cluster and images) :

- Browser: open `https://github.com/codespaces`, click **...** next to your Codespace and choose **Delete**
- Terminal:
```bash
gh codespace delete -c $CODESPACE_NAME
```

> Note: Push anything you want to keep with `git push` before deleting.

#
