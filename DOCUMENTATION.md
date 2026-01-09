## ✅ Python Installation Guide (Use **Python 3.12 ONLY**)

- **Install Python version: 3.11**
  - Versions **above 3.11** have limited ML library support
  - **Lower versions** may cause compatibility and dependency issues

---

### 🔗 Step 1: Open Official Python Download Page
Go to:
https://www.python.org/downloads/release/python-3127/

---

### 🪟 Step 2: Choose Correct Installer (Windows 64-bit)
- Scroll to the **Version section**
- Under **Windows**, download:
  - **Windows installer (64-bit)** → *Recommended*

---

### ⚙️ Step 3: Install Python
- Run the downloaded installer
- **IMPORTANT:**  
  ✔️ Tick **“Add Python to PATH environment variables”**  
- Continue installation with default settings

---

### 🧪 Step 4: Create Virtual Environment Using Python 3.11
Make sure your virtual environment is created using **Python 3.11**, not any other version.

This ensures:
- Proper ML library support  
- Fewer dependency issues  
- Stable development environment


---

## Dockerfile

You can **copy and paste it directly** into a file named `Dockerfile`.

```dockerfile
FROM python:3.11

WORKDIR /app

COPY . /app

RUN pip install --no-cache-dir -e .

EXPOSE 5000

ENV FLASK_APP=application.py

CMD ["python" , "application.py"]
```

## Kubernetes Deployment File

You can **copy and paste it directly** into a file named `k8s-deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vulnerable-flask
  namespace: vulnerable-test
  labels:
    app: vulnerable-flask
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vulnerable-flask
  template:
    metadata:
      labels:
        app: vulnerable-flask
    spec:
      securityContext:
        runAsUser: 0   # Running as root (vulnerability)
      containers:
        - name: flask-container
          image: flask-app:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
          securityContext:
            privileged: true                # Privileged container (vulnerability)
            allowPrivilegeEscalation: true
            readOnlyRootFilesystem: false
            runAsNonRoot: false
          env:
            - name: SECRET_PASSWORD
              value: "admin123"             # Hardcoded secret (vulnerability)
          volumeMounts:
            - name: host-root
              mountPath: /host              # Host path mount (vulnerability)
      volumes:
        - name: host-root
          hostPath:
            path: /                         # Host path mount (vulnerability)

---
apiVersion: v1
kind: Service
metadata:
  name: vulnerable-flask-service
  namespace: vulnerable-test
spec:
  type: NodePort
  selector:
    app: vulnerable-flask
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
      nodePort: 30081

```

---

## Insecure RBAC File

You can **copy and paste it directly** into a file named `insecure-rbac.yaml`.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: insecure-service-account
  namespace: vulnerable-test
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: insecure-binding
subjects:
- kind: ServiceAccount
  name: insecure-service-account
  namespace: vulnerable-test
roleRef:
  kind: ClusterRole
  name: cluster-admin  # Excessive permissions (vulnerability)
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: insecure-deployment
  namespace: vulnerable-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: insecure-app
  template:
    metadata:
      labels:
        app: insecure-app
    spec:
      serviceAccountName: insecure-service-account
      containers:
      - name: insecure-container
        image: alpine:latest
        command: ["sleep", "3600"]
        securityContext:
          runAsUser: 0
          capabilities:
            add: ["SYS_ADMIN", "NET_ADMIN"]  # Excessive capabilities

```

---

## KUBE-HUNTER File

You can **copy and paste it directly** into a file named `kube-hunter.yaml`.

```yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: kube-hunter
spec:
  template:
    metadata:
      labels:
        app: kube-hunter
    spec:
      containers:
        - name: kube-hunter
          image: aquasec/kube-hunter:0.6.8
          command: ["kube-hunter"]
          args: ["--pod"]
      restartPolicy: Never
```

---

## KUBE-BENCH File

You can **copy and paste it directly** into a file named `kube-bench.yaml`.

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: kube-bench
spec:
  hostPID: true
  containers:
    - name: kube-bench
      image: aquasec/kube-bench:latest
      args: ["--benchmark", "cis-1.23", "--json"]
      securityContext:
        privileged: true
      volumeMounts:
        - name: etc
          mountPath: /etc
          readOnly: true
        - name: var-lib
          mountPath: /var/lib
          readOnly: true
        - name: usr-bin
          mountPath: /usr/bin
          readOnly: true
        - name: usr-lib
          mountPath: /usr/lib
          readOnly: true
        - name: lib
          mountPath: /lib
          readOnly: true
        - name: proc
          mountPath: /proc
          readOnly: true
        - name: run-systemd
          mountPath: /run/systemd
          readOnly: true
  volumes:
    - name: etc
      hostPath:
        path: /etc
    - name: var-lib
      hostPath:
        path: /var/lib
    - name: usr-bin
      hostPath:
        path: /usr/bin
    - name: usr-lib
      hostPath:
        path: /usr/lib
    - name: lib
      hostPath:
        path: /lib
    - name: proc
      hostPath:
        path: /proc
    - name: run-systemd
      hostPath:
        path: /run/systemd
  restartPolicy: Never

```

---


## Python Script for Generating KubeBench Documentation

You can **copy and paste it directly** into a file named `kubebench-report-generator.py`.

```yaml

import json
from pathlib import Path

def load_kube_bench_results(json_path):
    with open(json_path, 'r') as file:
        return json.load(file)

def generate_report(data, output_path="kube_bench_report.md"):
    lines = []
    total_pass, total_fail, total_warn, total_info = 0, 0, 0, 0

    for control in data.get("Controls", []):
        lines.append(f"# Control: {control['text']} ({control['id']})")
        lines.append(f"**Node Type:** {control['node_type']}")
        lines.append("")

        for test in control.get("tests", []):
            section_header = f"## Section {test['section']}: {test['desc']}"
            lines.append(section_header)
            lines.append(f"- **Pass:** {test['pass']}")
            lines.append(f"- **Fail:** {test['fail']}")
            lines.append(f"- **Warn:** {test['warn']}")
            lines.append(f"- **Info:** {test['info']}")
            lines.append("")

            total_pass += test['pass']
            total_fail += test['fail']
            total_warn += test['warn']
            total_info += test['info']

            for result in test.get("results", []):
                lines.append(f"### {result['test_number']} - {result['test_desc']}")
                lines.append(f"- **Status:** {result['status']}")
                if result.get('reason'):
                    lines.append(f"- **Reason:** {result['reason'][:500]}{'...' if len(result['reason']) > 500 else ''}")
                if result.get('remediation'):
                    lines.append(f"- **Remediation:** {result['remediation'].replace(chr(10), ' ')}")
                lines.append("")

        lines.append("\n---\n")

    # Summary
    lines.append("# Summary")
    lines.append(f"- **Total Passed:** {total_pass}")
    lines.append(f"- **Total Failed:** {total_fail}")
    lines.append(f"- **Total Warnings:** {total_warn}")
    lines.append(f"- **Total Info:** {total_info}")
    lines.append("")

    with open(output_path, 'w') as file:
        file.write('\n'.join(lines))

    print(f" Report saved to {output_path}")

if __name__ == "__main__":
    json_input = "kube-bench-results.json"  # Adjust path if needed
    output_file = "kube_bench_report.md"
    data = load_kube_bench_results(json_input)
    generate_report(data, output_file)
```

---

## GCP Cloud Setup

---

### 1. Create a VM Instance on Google Cloud

1. Go to **Compute Engine → VM Instances**
2. Click **Create Instance**

**Basic Configuration**

* **Name:** `Whatever you want to name`
* **Machine Type:**

  * Series: **E2**
  * Preset: **Standard**
  * Memory: **16 GB RAM**
* **Boot Disk:**

  * Size: **150 GB**
  * Image: **Ubuntu 24.04 LTS**
* **Networking:**

  * Enable **HTTP** and **HTTPS** traffic and **Port Forwarding** turned on

Click **Create** to launch the instance.

---

### 2. Connect to the VM

* Use the **SSH** button in the Google Cloud Console to connect to the VM directly from the browser.

---

### 3. Configure the VM Instance

#### Clone the GitHub Repository

```bash
git clone https://github.com/data-guru0/TESTING-9.git ( Whatver your Github repo link )
ls
cd TESTING-9
ls
```

You should now see the project files inside the VM.

---

### 4. Install Docker

1. Open a browser and search for **“Install Docker on Ubuntu”**
2. Open the **official Docker documentation** (`docs.docker.com`)
3. Copy and paste the **first command block** into the VM terminal
4. Copy and paste the **second command block**
5. Test the Docker installation:

```bash
docker run hello-world
```

---

### 5. Run Docker Without `sudo`

From the same Docker documentation page, scroll to **Post-installation steps for Linux** and run **all four commands** one by one.

The last command is used to verify Docker works without `sudo`.

---

### 6. Enable Docker to Start on Boot

From the section **Configure Docker to start on boot**, run:

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

---

### 7. Verify Docker Setup

```bash
systemctl status docker
docker ps
docker ps -a
```

Expected results:

* Docker service shows **active (running)**
* No running containers
* `hello-world` container appears in exited state

---

### 8. Configure Minikube Inside the VM

#### Install Minikube

1. Search for **Install Minikube**
2. Open the official website: `minikube.sigs.k8s.io`
3. Select:

   * **OS:** Linux
   * **Architecture:** x86
   * **Installation Type:** Binary

Copy and run the installation commands provided on the website.

---

#### Start the Minikube Cluster

```bash
minikube start
```

Minikube uses **Docker internally**, which is why Docker was installed first.

---


---

### 9. Verify Kubernetes & Minikube Setup

```bash
minikube status
minikubr kubectl get nodes
minikube kubectl cluster-info
docker ps
```

Expected results:

* All Minikube components are running
* A single `minikube` node is visible
* Kubernetes cluster information is accessible
* Minikube container is running in Docker

---

### 10. Install kubectl

 - Search: `Install kubectl`
  - Instead of installing manually, go to the **Snap section** (below on the same page)

  ```bash
  sudo snap install kubectl --classic
  ```

  - Verify installation:

    ```bash
    kubectl version --client
    ```

### 11. Configure GCP Firewall (If Needed)

If Jenkins does not load, create a firewall rule:

* **Name:** `allow-apps`
* **Description:** Allow all traffic (for Jenkins demo)
* **Logs:** Off
* **Network:** default
* **Direction:** Ingress
* **Action:** Allow
* **Targets:** All instances
* **Source IP ranges:** `0.0.0.0/0`
* **Allowed protocols and ports:** All

---

## Enable Useful Minikube Addons

Enable the following Minikube addons to enhance cluster functionality:

```bash
minikube addons enable dashboard
minikube addons enable metrics-server
minikube addons enable ingress
```


## Install Required Dependencies

Update the system and install essential packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget apt-transport-https ca-certificates gnupg lsb-release python3-pip git
```

## Get Minikube IP

Retrieve the Minikube cluster IP and store it for later use:

```bash
minikube ip
```

## Create a new Namespace

Retrieve the Minikube cluster IP and store it for later use:

```bash
kubectl create namespace vulnerable-test
```

##  Build and Deploy Your App on VM

### Point Docker to Minikube
```bash
eval $(minikube docker-env)
````


### Build Docker Image

```bash
docker build -t flask-app:latest .
```


### Deploy Application to Kubernetes

```bash
kubectl apply -f k8s-deployment.yaml
```


### Check Pod Status

```bash
kubectl get pods -n vulnerable-test
```

> You will see the pods running.


### Expose the App (Port Forwarding)

```bash
kubectl port-forward svc/vulnerable-flask-service 5000:80 -n vulnerable-test --address 0.0.0.0

```

- Make sure to give correct service name.


### ✅ Access the Application

* Copy the **External IP**
* Open browser and visit:
- http://EXTERNAL-IP:5000

🎉 Your application is now live..
---


````md
## Run the kube-hunter Job

Apply the kube-hunter Job manifest:

```bash
kubectl create -f kube-hunter.yaml
````

## Find the Pod Name

You can find the pod created by the Job using either of the following commands:

```bash
kubectl get pods
```

## View the Test Results

Once you have the pod name, view the kube-hunter scan results from the pod logs:

```bash
kubectl logs <pod-name>
```

## Vulnerability Verifications

### Verify Privileged Container Access
Check whether the container can access the host filesystem:

```bash
kubectl get pods -n vulnerable-test
kubectl exec -it vulnerable-flask-6c76b9944b-qtvfd -n vulnerable-test -- ls /host

````

### Access Containers with Elevated Privileges

Open an interactive shell inside the container:

```bash
kubectl exec -it vulnerable-flask-6c76b9944b-qtvfd -n vulnerable-test -- /bin/bash
```

### Check Mounted Secrets

List secrets across all namespaces:

```bash
kubectl get secrets --all-namespaces
```

### Test Service Account Permissions

Verify what actions the service account is allowed to perform:

```bash
kubectl auth can-i --list --as=system:serviceaccount:vulnerable-test:insecure-service-account
```

## Apply kube-bench

Apply the kube-bench manifest to the cluster:

```bash
kubectl apply -f kube-bench.yaml
```

## View the kube-bench Report

Filter everything between `{` and `}` (the start and end of the JSON output) from the kube-bench logs and redirect it into a clean JSON file:

```bash
kubectl logs kube-bench | sed -n '/^{/,/^}$/p' > kube-bench-results.json
````

The output may look difficult to read in raw form, so it’s recommended to format and inspect the JSON using **jq**.

### What is jq?

`jq` is a powerful, lightweight, and flexible command-line JSON processor.
It works like `sed`, `awk`, or `grep`, but is specifically designed for JSON data.
You can use it to slice, filter, map, and transform structured JSON easily.

### Install jq

```bash
sudo apt install jq
```

### Pretty-print the JSON Report

```bash
jq . kube-bench-results.json
```




## Generate kube-bench Report

Run the Python script to generate the kube-bench report:

```bash
python3 kubebench-report-generator.py
```