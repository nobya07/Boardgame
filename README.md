# 🎲 Boardgame — DevSecOps CI/CD Project

A complete end-to-end CI/CD pipeline for a Spring Boot application, automated from code push to Kubernetes deployment.

---

## 🏗️ Pipeline Architecture

```
GitHub → Jenkins → SonarQube → Nexus → Docker → Trivy → DockerHub → Kubernetes
```

| Step | Tool | What Happens |
|------|------|-------------|
| 1 | **GitHub** | Code push triggers the pipeline |
| 2 | **Jenkins** | Checks out code, runs Maven build |
| 3 | **SonarQube** | Static code analysis + quality gate |
| 4 | **Nexus** | Stores build artifact (JAR) |
| 5 | **Docker** | Builds container image |
| 6 | **Trivy** | Scans image for vulnerabilities |
| 7 | **DockerHub** | Pushes verified image to registry |
| 8 | **Kubernetes** | Deploys with zero-downtime rolling update |

---

## 🖥️ Infrastructure

| Component | Count | Details |
|-----------|-------|---------|
| Kubernetes Master | 1x EC2 | Control plane (API server, scheduler, etcd) |
| Kubernetes Worker | 2x EC2 | Runs application pods |
| Jenkins | Docker | CI/CD orchestration |
| SonarQube | Docker | Code quality analysis — port `9000` |
| Nexus | Docker | Artifact repository — port `8081` |

---

## ⚙️ Jenkins Setup

### 1. Install Plugins
> Manage Jenkins → Plugins → Available Plugins

| Plugin | Purpose |
|--------|---------|
| Pipeline | Jenkinsfile-based declarative pipelines |
| Git / GitHub | Clone code from GitHub |
| Docker Pipeline | Build and push Docker images |
| Kubernetes CLI | Run `kubectl` commands in pipeline |
| SonarQube Scanner | Code analysis integration |
| Config File Provider | Store Maven `settings.xml` with Nexus creds |
| Maven Integration | Run Maven builds |
| Email Extension | Send build notifications |
| Credentials Binding | Safely inject secrets into pipeline |

### 2. Configure Tools
> Manage Jenkins → Global Tool Configuration

- **JDK17** — set JAVA_HOME to Java 17 path
- **Maven3** — point to Maven 3.x install directory
- **Sonar Scanner** — configure SonarQube Scanner
- **Docker** — point to Docker binary

### 3. Add Credentials
> Manage Jenkins → Credentials → Global

| Credential ID | Type | Value |
|---------------|------|-------|
| `docker-cred` | Username/Password | DockerHub login |
| `sonar-token` | Secret Text | Token from SonarQube UI |
| `k8s-token` | Secret Text | Kubernetes service account token |
| `github-cred` | Username/Password | GitHub username + PAT |

---

## 🔍 SonarQube Setup

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

1. Access at `http://<your-ip>:9000` (default: `admin` / `admin`)
2. Create a project and generate an auth token
3. Add token to Jenkins as `sonar-token`
4. Add this webhook so Jenkins knows when quality gate passes:

```
http://<jenkins-ip>:8080/sonarqube-webhook/
```

> ⚠️ Without the webhook, the pipeline will hang forever at the Quality Gate step.

---

## 📦 Nexus Setup

```bash
docker run -d --name Nexus -p 8081:8081 sonatype/nexus3
```

1. Access at `http://<your-ip>:8081`
2. Create two Maven repositories:
   - `maven-releases` — for release versions (e.g. `1.0.0`)
   - `maven-snapshots` — for snapshot versions (e.g. `0.0.5-SNAPSHOT`)
3. Update `pom.xml` to point to these Nexus URLs
4. Add Nexus credentials via Config File Provider plugin in Jenkins

---

## 🔐 Kubernetes RBAC

Jenkins needs permission to deploy to Kubernetes. Apply these 4 files in the `webapps` namespace:

| File | Purpose |
|------|---------|
| `svc.yaml` | Creates `jenkins-sa` ServiceAccount in `webapps` namespace |
| `role.yaml` | Creates Role with pod/deployment/service permissions |
| `bind.yaml` | Binds Role to `jenkins-sa` (RoleBinding) |
| `sec.yaml` | Creates Secret to store the long-lived token |

**Extract the token for Jenkins:**
```bash
kubectl get secret <secret-name> -n webapps \
  -o jsonpath='{.data.token}' | base64 --decode
```

Add this token to Jenkins as credential ID `k8s-token`.

---

## 🚀 Kubernetes Deployment

### Deployment Config

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: boardgame-deployment
  namespace: webapps
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: boardgame
  template:
    metadata:
      labels:
        app: boardgame
    spec:
      containers:
        - name: boardgame
          image: gajendra1/boardshack:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"

          # Required — Spring Boot takes ~30s to start
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 5

          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
---
apiVersion: v1
kind: Service
metadata:
  name: boardgame-ssvc
  namespace: webapps
spec:
  type: NodePort
  selector:
    app: boardgame
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30007
```

> ⚠️ **CrashLoopBackOff fix**: The probes above are critical. Without them, Kubernetes sends traffic before Spring Boot finishes starting and kills the pod (Exit Code 143).
>
> If `/actuator/health` is not in your app, replace `httpGet` probes with:
> ```yaml
> tcpSocket:
>   port: 8080
> ```

### Rolling Update Settings

| Setting | Value | Meaning |
|---------|-------|---------|
| `replicas` | 3 | 3 pods always running |
| `maxUnavailable` | 1 | Max 1 pod down during update |
| `maxSurge` | 1 | Max 1 extra pod during update |
| `nodePort` | 30007 | Access via `http://<node-ip>:30007` |

---

## ✅ Validation Commands

```bash
# Check all pods are Running with 0 restarts
kubectl get pods -n webapps

# Confirm NodePort 30007 is assigned
kubectl get svc -n webapps

# Detailed info + events for a crashing pod
kubectl describe pod <pod-name> -n webapps

# View Spring Boot startup logs
kubectl logs <pod-name> -n webapps

# Check all nodes are Ready
kubectl get nodes

# Check Jenkins / SonarQube / Nexus containers
docker ps
```

---

## 📧 Email Notifications (Pipeline Success / Failure)

Get an email automatically whenever your Jenkins pipeline succeeds or fails.  
Everything is configured from the Jenkins UI — no code changes needed except adding a `post {}` block to your Jenkinsfile.

---

### Step 1 — Generate Gmail App Password

> You must use an **App Password**, not your regular Gmail password.

1. Go to **myaccount.google.com → Security**
2. Enable **2-Step Verification** (mandatory before App Passwords appear)
3. Search for **App Passwords** in the search bar at the top
4. Under **Select app** choose **Mail**
5. Under **Select device** choose **Other** → type `Jenkins`
6. Click **Generate**
7. Copy the **16-character password** shown (e.g. `abcd efgh ijkl mnop`)

> ⚠️ This password is shown **only once**. Copy it before closing the popup.

---

### Step 2 — Add Credential in Jenkins UI

```
Jenkins Dashboard
  └── Manage Jenkins
        └── Credentials
              └── System
                    └── Global credentials (unrestricted)
                          └── + Add Credentials
```

Fill in the form:

| Field | Value |
|-------|-------|
| Kind | `Username with password` |
| Scope | `Global` |
| Username | `your-email@gmail.com` |
| Password | the 16-character App Password from Step 1 |
| ID | `gmail-cred` |
| Description | `Gmail App Password for Jenkins notifications` |

Click **Create**.

---

### Step 3 — Configure E-mail Notification (Basic SMTP)

```
Jenkins Dashboard
  └── Manage Jenkins
        └── System
              └── scroll down to → E-mail Notification
```

Fill in the fields:

| Field | Value |
|-------|-------|
| SMTP Server | `smtp.gmail.com` |
| Default user e-mail suffix | `@gmail.com` |

Click **Advanced** button and fill in:

| Field | Value |
|-------|-------|
| Use SMTP Authentication | ✅ check this |
| User Name | `your-email@gmail.com` |
| Password | App Password from Step 1 |
| Use SSL | ✅ check this |
| SMTP Port | `465` |
| Reply-To Address | `your-email@gmail.com` |

At the bottom of this section:
- Enter your email in **Test e-mail recipient**
- Click **Test configuration**
- You should receive a test email ✅

Click **Save**.

---

### Step 4 — Configure Extended E-Mail Notification

```
Jenkins Dashboard
  └── Manage Jenkins
        └── System
              └── scroll down to → Extended E-mail Notification
```

Fill in the fields:

| Field | Value |
|-------|-------|
| SMTP Server | `smtp.gmail.com` |
| SMTP Port | `465` |
| Credentials | select `gmail-cred` (created in Step 2) |
| Use SSL | ✅ check this |
| Default Content Type | `HTML (text/html)` |
| Default Recipients | `your-email@gmail.com` |
| Default Subject | `$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!` |

Scroll down to **Default Content** and paste:

```
$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:

Check console output at $BUILD_URL to view the results.
```

Click **Save**.

---

### Step 5 — Add Post Block to Jenkinsfile

Add this `post {}` block at the end of your `Jenkinsfile` pipeline:

```groovy
post {
    always {
        script {
            def jobName    = env.JOB_NAME
            def buildNum   = env.BUILD_NUMBER
            def pipeStatus = currentBuild.result ?: 'UNKNOWN'
            def bodyColor  = pipeStatus == 'SUCCESS' ? '#27AE60' : '#E74C3C'
            def statusIcon = pipeStatus == 'SUCCESS' ? '✅' : '❌'

            emailext(
                subject: "${statusIcon} Pipeline ${pipeStatus}: ${jobName} #${buildNum}",
                body: """
                    <html><body style="font-family:Arial,sans-serif;">
                    <h2 style="color:${bodyColor};">${statusIcon} Build ${pipeStatus}</h2>
                    <table style="border-collapse:collapse;width:100%;">
                      <tr><td style="padding:8px;border:1px solid #ddd;"><b>Job</b></td>
                          <td style="padding:8px;border:1px solid #ddd;">${jobName}</td></tr>
                      <tr><td style="padding:8px;border:1px solid #ddd;"><b>Build #</b></td>
                          <td style="padding:8px;border:1px solid #ddd;">${buildNum}</td></tr>
                      <tr><td style="padding:8px;border:1px solid #ddd;"><b>Status</b></td>
                          <td style="padding:8px;border:1px solid #ddd;color:${bodyColor};"><b>${pipeStatus}</b></td></tr>
                      <tr><td style="padding:8px;border:1px solid #ddd;"><b>URL</b></td>
                          <td style="padding:8px;border:1px solid #ddd;">
                            <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></td></tr>
                    </table>
                    <p style="color:#888;font-size:12px;">Sent by Jenkins — Boardgame CI/CD Pipeline</p>
                    </body></html>
                """,
                to: 'your-email@gmail.com',
                from: 'your-email@gmail.com',
                replyTo: 'your-email@gmail.com',
                mimeType: 'text/html'
            )
        }
    }
}
```

> Replace `your-email@gmail.com` with your actual Gmail address.

---

### What the Email Looks Like

| Field | Example |
|-------|---------|
| Subject (success) | `✅ Pipeline SUCCESS: boardgame-pipeline #42` |
| Subject (failure) | `❌ Pipeline FAILURE: boardgame-pipeline #43` |
| Body | HTML table — job name, build number, status, direct link to build logs |

---

### Common Email Issues

| Problem | Fix |
|---------|-----|
| Test email not received | Check spam folder. Verify App Password is correct |
| `AuthenticationFailedException` | Re-generate App Password. Make sure 2FA is enabled on Gmail |
| `Connection refused` on port 465 | Make sure **Use SSL** is checked and port is `465` not `587` |
| Email sent but pipeline not triggering it | Make sure `emailext` plugin (Email Extension) is installed |

---

## 🐛 Troubleshooting

| Error | Symptom | Fix |
|-------|---------|-----|
| `CrashLoopBackOff` | Pods keep restarting | Add startup/readiness/liveness probes. Check `kubectl logs <pod> -n webapps` |
| `Docker not found` | Jenkins build fails | Install Docker on host. Run: `usermod -aG docker jenkins` |
| `Trivy not found` | Scan stage fails | Install: `apt install trivy` or use official installer |
| `Sonar auth error` | Analysis rejected | Re-generate token in SonarQube UI, update `sonar-token` credential |
| `Quality gate stuck` | Pipeline hangs | Set webhook in SonarQube: `http://<jenkins-ip>:8080/sonarqube-webhook/` |
| `NodePort timeout` | Can't reach port 30007 | Add inbound rule for port `30007` in EC2 Security Group |
| `TLS timeout` | HTTPS fails | Check certificate validity and Ingress TLS annotations |
| `Git auth issues` | Can't clone repo | Add GitHub PAT to Jenkins credentials |
| `ErrImagePull` | Pod can't pull image | Check image name/tag on DockerHub. Verify `docker-cred` secret in `webapps` namespace |

### Diagnosing a crashing pod — run in this order:

```bash
# 1. Get pod name
kubectl get pods -n webapps

# 2. Check logs for error message
kubectl logs <pod-name> -n webapps

# 3. Check Exit Code in "Last State" section
kubectl describe pod <pod-name> -n webapps
```

| Exit Code | Meaning |
|-----------|---------|
| `0` | App exited cleanly — check Spring context failure |
| `1` | App crashed — check logs for exceptions |
| `137` | OOMKilled — increase memory limits |
| `143` | Killed by Kubernetes — add health probes |