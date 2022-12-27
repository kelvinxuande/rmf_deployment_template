# RMF Deployment Template
This branch contains the bringup instructions and configurations to setup a Kubernetes cluster infrastructure.
This deployment is fully configurable and minimally will need following edits prior to bringup. Be sure to edit these prior to running through the next steps.
- RMF configuration in `rmf-deployment/`
    - `rmf_server_config.py` - replace DNS name `rmf-deployment-template.open-rmf.org` with your own.
    - `values.yaml` - replace registryUrl `ghcr.io/open-rmf` and DNS name `rmf-deployment-template.open-rmf.org` with your own.
    - `rmf-site-modules.yaml` - Add site specific nodes (e.g. fleet and door adapters) to the template.
    - `cyclonedds.xml` - if you are using cyclonedds to communicate across multiple nodes on different machines, update the `Peers` in the `.xml`.
    - If you are using ros `galactic`, please switch the corresponding cyclonedds config path to `cyclonedds_galactic.xml`.

## Provisioning
We will need the following resources:
* 1 Cloud VM ( For RMF k8s cluster )
* Operator Machine ( For provisioning ) with SSH access to this VM.

### Cloud resource provisioning

#### Provision Cloud VM
VM specifications:
- 4 vCPU, 8GB memory _[recommended size for 2 robots, 1 door, 1 lift (elevator), your mileage may vary based on your scale]_
- 80GB SSD storage
- 1 public IP
- Ports 443, 22 open

#### Add DNS record
Add VM public IP to domain url in DNS records (eg. Route 53 in AWS)

### Specifications Tested (on Amazon Web Services)

If using Amazon Linux image, replace `ubuntu` in commands with `ec2-user`
- AMI: ubuntu server 20.04
- Instance Type: t2.xlarge (4 vCPUs)
- Configure root volume to 80 GB. Note that we can also [provision, attach and boot from EBS](https://www.youtube.com/watch?v=pKA46UVy874) here, instead.
- Remember to (provision and) set keypairs and security group for newly created EC2 instance

Useful references:
- [Amazon EC2 Basics & Instances Tutorial](https://www.youtube.com/watch?v=iHX-jtKIVNA)
- [SSH to EC2 Instances using Linux or Mac Tutorial](https://www.youtube.com/watch?v=8UqtMcX_kg0)

## Remember to target the VM Public IP v4 address with domain URL in DNS Records
    - E.g. Route 53 > Hosted zones > test-app-domain.com

## Do the following locally

#### Set up SSH
- Restrict permissions of local pem file (only readable and writer by owner, required)
`sudo chmod 0600 siot_test-rmf_ec2-keypair.pem` 
- SSH into newly provisioned EC2 instance
```
# ssh -i siot_test-rmf_ec2-keypair.pem ec2-user@<insert-ip>
ssh -i siot_test-rmf_ec2-keypair.pem ubuntu@<insert-ip>
```

#### Install Docker
```
# For Amazon Linux Image:
# sudo yum update -y
# sudo yum -y install docker

# For Ubuntu Image:
sudo apt update && sudo apt install docker.io -y

# Start Docker service
sudo service docker start

# verify that docker service is running
cat /var/run/docker.pid

# Add ec2-user/ Ubuntu user to Docker Group to access commands
# sudo usermod -a -G docker ec2-user
sudo usermod -a -G docker ubuntu

# log out, log back in and run
docker version
```
Useful references:
- [Installing Docker on Amazon Linux 2](https://stackoverflow.com/questions/53918841/how-to-install-docker-on-amazon-linux2)
- [Verify that Docker service is running](https://www.howtogeek.com/devops/how-to-check-if-the-docker-daemon-or-a-container-is-running/#:~:text=Another%20way%20to%20check%20for,and%20ready%20for%20CLI%20connections)

#### Install k3s via [k3sup](https://github.com/alexellis/k3sup)
```
# original comamnd
# curl -sLS https://get.k3sup.dev | sh

curl -sfL https://get.k3sup.dev | sh -s - --write-kubeconfig-mode 644

sudo install k3sup /usr/local/bin/
```
Run modified commands to deal with:
- Fatal depreciation of `no-deploy` flag, [see issue](https://github.com/k3s-io/k3s/issues/1160).
- level=fatal msg="invalid node-ip: invalid ip format ''"
```
# original command:
# k3sup install --local --user ubuntu --cluster --k3s-extra-args '--flannel-iface=ens5 --no-deploy traefik --write-kubeconfig-mode --docker'

k3sup install --local --user ubuntu --cluster --k3s-extra-args '--disable traefik --write-kubeconfig-mode --docker'

# Generic troubleshooting/ k3s install
# https://www.loggly.com/ultimate-guide/using-journalctl/
journalctl -f
```

#### Setup and apply base Kubernetes Deployment
```
git clone -b cloud_infra https://github.com/open-rmf/rmf_deployment_template.git
cd rmf_deployment_template

# Change file permissions to deal with:
# WARN[0000] Unable to read /etc/rancher/k3s/k3s.yaml, please start server with --write-kubeconfig-mode to modify kube config permissions 
# error: error loading config file "/etc/rancher/k3s/k3s.yaml": open /etc/rancher/k3s/k3s.yaml: permission denied
sudo chmod 755 /etc/rancher/k3s/k3s.yaml

kubectl apply -f infrastructure/nginx/ingress-nginx.yaml
kubectl wait --for=condition=available deployment/ingress-nginx-controller -n ingress-nginx --timeout=2m

# listing all running pods on instance
# https://kubernetes.io/docs/tasks/access-application-cluster/list-all-running-container-images/
kubectl get pods --all-namespaces

# validating connection to k3s control
# https://discuss.kubernetes.io/t/the-connection-to-the-server-host-6443-was-refused-did-you-specify-the-right-host-or-port/552
kubectl cluster-info

# Get ingress IP
kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath="{.status.loadBalancer.ingress[0].ip}"

# Get the Node-Pod mapping
kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --all-namespaces
```

#### Setup SSL Certificates
```
kubectl create namespace cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=available deployment/cert-manager -n cert-manager --timeout=2m

# NOTE: Specify your `ACME_EMAIL` and `DOMAIN_NAME` for letsencrypt-issuer-production
# test-app-domain.com
# kelvinxuande@gmail.com
export DOMAIN_NAME=rmf-deployment-template.open-rmf.org
export ACME_EMAIL=YOUREMAIL@DOMAIN.com
envsubst < infrastructure/cert-manager/letsencrypt-issuer-production.yaml | kubectl apply -f -
kubectl get certificates # should be true, might need to wait a minute
# if it remains false i.e. after 3-5 mins, check that the VM Public IP v4 address is targetted with domain URL in DNS Records
```

#### Setup and apply argoCD Deployment
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl create namespace deploy
```

#### Port-forwarding via SSH
Note: To cater to the non-SSL visit in this section (for testing only), enable port 80 http on Security Group
```
# Get the initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# keep open
kubectl port-forward svc/argocd-server -n argocd 9090:443

# open a new terminal and SSH to instance 
ssh -i siot_test-rmf_ec2-keypair.pem -L 9090:localhost:9090 ubuntu@<insert-ip>
```
On browser, visit localhost:9090 with `admin` username and password acquired above.

#### argoCD setup
- Connect the repository. View the docs to learn how to configure ArgoCD
- When adding a "new app" on argocd, specify the [repository](https://github.com/open-rmf/rmf_deployment_template.git), setup to track `cloud_infra` branch and `rmf_deployment` directory.
- Use dropdown hints during this process whenever possible. Be sure to fill in the namespace parameter!

Now if you sync the app, we should see the full deployment "come alive".

#### Add domain url and initial credentials (after keycloak pod is running)
Note that for this particular repository/ branch; keycloak has been omitted
```
cd infrastructure
# ./keycloak-setup.bash rmf-deployment-template.open-rmf.org
./keycloak-setup.bash test-app-domain.com
```
RMF web dashboard will now be accessible on your_url/dashboard (eg. rmf.open-rmf.org/dashboard); users can be manage via your_url/auth (eg. rmf.open-rmf.org/auth). In this case, use `test-app-domain.com/dashboard` and `test-app-domain.com/auth`
