# CI/CD와 PLG 실습 과정

---

## 사전 작업 - Terraform 인프라 구성

Terraform을 사용하여 인프라를 코드로 관리하고 자동화합니다.

- GitHub Repository: [Terraform Basic](https://github.com/kangbock/terraform-basic)
<br><br>

---

## CI/CD 

지속적 통합과 배포 자동화를 위한 파이프라인을 Jenkins, Kaniko, Harbor, ArgoCD로 구성합니다.
<br>

**Workflow**
![alt text](img/image.png)
<br>

- Jenkins로 소스코드 통합 빌드
- Kaniko로 Docker 이미지 생성 후 Harbor에 이미지 푸시
- ArgoCD가 변경된 배포 YAML을 통해 자동 배포 수행
- Slack으로 상태 알림
<br><br>

---

## PLG

애플리케이션과 인프라의 메트릭 및 로그를 수집하여 관찰성을 높이고 분석합니다.
<br>

### Prometheus Workflow

메트릭 수집 및 알림 발송
<br>

![alt text](img/image-2.png)
<br>

### Grafana Loki Workflow

로그 수집 및 분석
<br>

![alt text](img/image-3.png)
<br><br>

---

## Azure Login

Azure VM(Ubuntu)에서 Docker, Azure CLI 및 Kubernetes CLI 설정 및 AKS 연결
<br>

**Azure VM (Ubuntu 22.04)**
```
sudo apt update 
sudo apt-get upgrade -y
sudo apt-get install -y docker.io 
sudo git clone https://github.com/kangbock/msa_nginx.git 
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login --service-principal --username $APPID --password $APPPW --tenant $TENANT --output table
az account set --subscription kblee_cc1_gp_mpn_2-cloudsecurity-02
az aks install-cli
az aks get-credentials --resource-group poc-rg --name poc-test-aks
```
<br>

**Helm Install**

Helm을 이용하여 Kubernetes 앱 배포 및 관리
<br>

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
<br><br>

---

## Cert-manager

인증서 자동 발급 및 관리 (LetsEncrypt 연동)
<br>

```
kubectl create namespace cert-manager

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version=v1.8.0 \
  --set installCRDs=true \
  --set nodeSelector."kubernetes\.io/os"=linux
```
<br><br>

---

## Harbor

프라이빗 컨테이너 레지스트리 구축

- NGINX Ingress Controller 및 SSL 적용
- Cert-manager로 자동 인증서 관리
<br>

```
kubectl create ns devops-tools
kubectl apply -f R-D/harbor/harbor-certificate.yaml
kubectl get nodes --show-labels | grep linux
```
<br>

**Cluster Issuer**
```
kubectl apply -f cert-manager/.
```
<br>

**Harbor Deploy**
```
# nginx ingress controller install
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

# repo 등록
helm repo add harbor https://helm.goharbor.io

# 압축파일 다운로드
helm fetch harbor/harbor --untar

# env
sed -i 's/core.harbor.domain/harbor.k-tech.cloud/g' ./harbor/values.yaml
sed -i 's/className: ""/className: "nginx"/g' ./harbor/values.yaml
sed -i '19 s/certSource: auto/certSource: secret/g' ./harbor/values.yaml
sed -i '28 s/secretName: ""/secretName: harbor-secret/g' ./harbor/values.yaml
sed -i '/# for Envoy/a\      certmanager.k8s.io/disable-auto-restart: "true"' ./harbor/values.yaml
sed -i '/# for Envoy/a\      cert-manager.io/cluster-issuer: letsencrypt' ./harbor/values.yaml
# sed -i '/# for Envoy/a\      cert-manager.io/cluster-issuer: letsencrypt-staging' ./harbor/values.yaml

# harbor deploy
helm install harbor -f ./harbor/values.yaml ./harbor/. -n devops-tools

# Harbor login
# ID : admin
# PW : Harbor12345
```
<br><br>

---

## Istio Service Mesh

서비스 간 통신 보안과 관리

- 자동 사이드카 주입
- Kiali를 통한 가시성 확보
<br>

**Isito Download**
```
curl -L https://istio.io/downloadIstio | sh -
mv istio-1.24.2 istio
cd istio
export PATH=$PWD/bin:$PATH
cd ..
```
<br>

**Istio Deploy**
```
istioctl install --set profile=demo -y
# istioctl install --set profile=demo --skip-confirmation
```
<br>

**auto sidecar injection**
```
kubectl label namespace default istio-injection=enabled --overwrite
kubectl label namespace istio-system istio-injection=enabled --overwrite
kubectl label namespace monitoring istio-injection=enabled --overwrite

kubectl get namespace -L istio-injection
```
<br>

**sidecar check**
```
istioctl experimental check-inject <pod-name>
```
<br>

### Kiali

**kiali.yaml**
```
data:
  config.yaml: |
    external_services:
      prometheus:
        custom_metrics_url: http://kube-prometheus-stack-prometheus.monitoring:9090
        url: http://kube-prometheus-stack-prometheus.monitoring:9090
      grafana:
        custom_metrics_url: http://kube-prometheus-stack-grafana.monitoring:80
        url: http://kube-prometheus-stack-grafana.monitoring:80
```
<br>

```
kubectl apply -f istio/samples/addons/kiali.yaml
kubectl apply -f R-D/kiali/.
```
<br><br>

---

## Jenkins

**Deploy**
```
kubectl apply -f R-D/jenkins/.
```
<br>

**Password**
```
kubectl exec -it svc/jenkins-service -n devops-tools -- cat /var/jenkins_home/secrets/initialAdminPassword
```
<br>

### Kaniko

컨테이너 이미지 빌드를 위한 보안 강화 도구

- Docker 데몬 없이 Kubernetes 내부 빌드 가능
<br>

#### Docker vs Kaniko
**Docker** : Docker는 Docker 데몬이 호스트 시스템에서 실행되고 이미지를 빌드하는 데몬 기반 접근 방식을 사용합니다. 이를 위해서는 특히 Kubernetes 클러스터에서 보안 문제가 될 수 있는 권한 있는 액세스가 필요합니다.

**Kaniko** : Kaniko는 컨테이너 또는 Kubernetes 클러스터 내부의 Dockerfile에서 컨테이너 이미지를 빌드하는 도구입니다. 특별한 권한이 필요하지 않으므로 Kubernetes 환경의 보안이 더욱 강화됩니다.
<br>

**Create secret**
```
docker login https://harbor.k-tech.cloud
cat ~/.docker/config.json
cat ~/.docker/config.json | base64
```
<br>

**regcred.yaml**
```
apiVersion: v1
kind: Secret
metadata:
  name: docker-config-secret
  namespace: default
data:
  .dockerconfigjson: 인코딩한 데이터
type: kubernetes.io/dockerconfigjson
```

```
kubectl apply -f R-D/kaniko/.
kubectl apply -n devops-tools -f R-D/kaniko/.
kubectl apply -n istio-system -f R-D/kaniko/.
```
<br>

### Jenkins Pipeline 구성

CI/CD 자동화를 위한 Jenkins Pipeline

- Kaniko를 이용한 컨테이너 이미지 빌드
- GitOps를 통한 자동 배포
- Slack 알림 연동
<br>

**Configuration**
<aside class="warning">💡 check : Do not allow the pipeline to resume if the controller restarts</aside><br>

<aside  class="warning">💡 check : GitHub hook trigger for GITScm polling</aside><br><br>

**Frontend**
```
podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
    spec:
      containers:
      - name: alpine
        image: alpine/git:latest
        command:
        - sleep
        args:
        - 99d
      - name: kaniko
        image: gcr.io/kaniko-project/executor:debug
        command:
        - sleep
        args:
        - 99d
        volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
      restartPolicy: Never
      volumes:
      - name: kaniko-secret
        secret:
          secretName: docker-config-secret
          items:
            - key: .dockerconfigjson
              path: config.json
''') {

    node(POD_LABEL) {
        try {
            stage("Set Variable") {
                git url: 'https://github.com/kangbock/msa_nginx.git', branch: 'main'
                script(){
                    env.GIT_COMMIT = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    GIT_TAG = sh (script: 'git describe --always', returnStdout: true).trim();
                    SLACK_CHANNEL = "#devops";
                    SLACK_SUCCESS_COLOR = "#2C953C";
                    SLACK_FAIL_COLOR = "#FF3232";
                    // Git Commit 계정
                    GIT_COMMIT_AUTHOR = sh(script: "git --no-pager show -s --format=%an ${env.GIT_COMMIT}", returnStdout: true).trim();
                    // Git Commit 메시지
                    GIT_COMMIT_MESSAGE = sh(script: "git --no-pager show -s --format=%B ${env.GIT_COMMIT}", returnStdout: true).trim();
                }
            }
            slackSend (
                channel: SLACK_CHANNEL,
                color: SLACK_SUCCESS_COLOR,
                message: "========================================\n The ${env.JOB_NAME}(${env.BUILD_NUMBER}) pipeline has started.\n\n          Author : ${GIT_COMMIT_AUTHOR} \n          Commit Message : ${GIT_COMMIT_MESSAGE}\n\n${env.BUILD_URL}"
            )
        } catch(git) {
                slackSend (
                    channel: SLACK_CHANNEL,
                    color: SLACK_SUCCESS_COLOR,
                    message: "========================================\n${env.JOB_NAME}(${env.BUILD_NUMBER}) pipeline failed.\n\n          Author : ${GIT_COMMIT_AUTHOR} \n          Commit Message : ${GIT_COMMIT_MESSAGE}\n\n${env.BUILD_URL}"
                )
            throw git;
        }
        

        stage('Kaniko Build and Push') {
            git url: 'https://github.com/kangbock/msa_nginx.git', branch: 'main'
            
            container('kaniko') {
                try {
                    stage('nginx') {
                        sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --cache=true --destination=harbor.k-tech.cloud/msa/nginx:${BUILD_NUMBER}'
                    }
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_SUCCESS_COLOR,
                        message: "NGINX Image Deployment and Push Successfully."
                    )
                } catch(Build) {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_FAIL_COLOR,
                        message: "Image deployment and push failed."
                    )
                    throw Build;
                }
            }
            
            git url: 'https://github.com/kangbock/msa_deploy.git', branch: 'main'
            container('alpine') {
                try {
                    stage('GitHub Push') {
                        sh 'sed -i s/nginx:.*/nginx:${BUILD_NUMBER}/g ./nginx.yaml'
		                    sh 'git config --global --add safe.directory /home/jenkins/agent/workspace/nginx'
		                    sh 'git config --global user.email \'kangbock@naver.com\''
		                    sh 'git config --global user.name \'kangbock\''
		                    sh 'git config --global --add safe.directory /home/jenkins/agent/workspace/main'
		                    sh 'git add ./'
		                    sh 'git commit -a -m "updated the image tag to ${BUILD_NUMBER}" || true'
		                    sh 'git remote add kb97 https://<GIT_TOKEN>@github.com/kangbock/msa_deploy.git'
		                    sh 'git push -u kb97 main'
                    }
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_SUCCESS_COLOR,
                        message: "Deployment was successful.\n========================================"
                    )
                } catch(push) {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_FAIL_COLOR,
                        message: "Deployment failed.\n========================================"
                    )
                    throw push;
                }
            }
        }
    }
}
```
<br>

**Backend (Login)**
```
podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    metadata:
      name: login
    spec:
      containers:
      - name: alpine
        image: alpine/git:latest
        command:
        - sleep
        args:
        - 99d
      - name: kaniko
        image: gcr.io/kaniko-project/executor:debug
        command:
        - sleep
        args:
        - 99d
        volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
      restartPolicy: Never
      volumes:
      - name: kaniko-secret
        secret:
          secretName: docker-config-secret
          items:
            - key: .dockerconfigjson
              path: config.json
''') {

    node(POD_LABEL) {
        try {
            stage("Set Variable") {
                git url: 'https://github.com/kangbock/msa_loginjs.git', branch: 'main'
                script(){
                    env.GIT_COMMIT = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    GIT_TAG = sh (script: 'git describe --always', returnStdout: true).trim();
                    SLACK_CHANNEL = "#devops";
                    SLACK_SUCCESS_COLOR = "#2C953C";
                    SLACK_FAIL_COLOR = "#FF3232";
                    // Git Commit 계정
                    GIT_COMMIT_AUTHOR = sh(script: "git --no-pager show -s --format=%an ${env.GIT_COMMIT}", returnStdout: true).trim();
                    // Git Commit 메시지
                    GIT_COMMIT_MESSAGE = sh(script: "git --no-pager show -s --format=%B ${env.GIT_COMMIT}", returnStdout: true).trim();
                }
            }
            slackSend (
                channel: SLACK_CHANNEL,
                color: SLACK_SUCCESS_COLOR,
                message: "========================================\n The ${env.JOB_NAME}(${env.BUILD_NUMBER}) pipeline has started.\n\n          Author : ${GIT_COMMIT_AUTHOR} \n          Commit Message : ${GIT_COMMIT_MESSAGE}\n\n${env.BUILD_URL}"
            )
        } catch(git) {
                slackSend (
                    channel: SLACK_CHANNEL,
                    color: SLACK_SUCCESS_COLOR,
                    message: "========================================\n${env.JOB_NAME}(${env.BUILD_NUMBER}) pipeline failed.\n\n          Author : ${GIT_COMMIT_AUTHOR} \n          Commit Message : ${GIT_COMMIT_MESSAGE}\n\n${env.BUILD_URL}"
                )
            throw git;
        }
        

        stage('Kaniko Build and Push') {
            git url: 'https://github.com/kangbock/msa_loginjs.git', branch: 'main'
            
            container('kaniko') {
                try {
                    stage('login') {
                        sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --cache=true --destination=harbor.k-tech.cloud/msa/login_js:${BUILD_NUMBER}'
                    }
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_SUCCESS_COLOR,
                        message: "LOGIN Image Deployment and Push Successfully."
                    )
                } catch(Build) {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_FAIL_COLOR,
                        message: "Image deployment and push failed."
                    )
                    throw Build;
                }
            }
            
            git url: 'https://github.com/kangbock/msa_deploy.git', branch: 'main'
            container('alpine') {
                try {
                    stage('GitHub Push') {
                        sh 'sed -i s/login_js:.*/login_js:${BUILD_NUMBER}/g ./login_js.yaml'
		                    sh 'git config --global --add safe.directory /home/jenkins/agent/workspace/login'
		                    sh 'git config --global user.email \'kangbock@naver.com\''
		                    sh 'git config --global user.name \'kangbock\''
		                    sh 'git config --global --add safe.directory /home/jenkins/agent/workspace/main'
		                    sh 'git add ./'
		                    sh 'git commit -a -m "updated the image tag to ${BUILD_NUMBER}" || true'
		                    sh 'git remote add kb97 https://<GIT_TOKEN>@github.com/kangbock/msa_deploy.git'
		                    sh 'git push -u kb97 main'
                    }
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_SUCCESS_COLOR,
                        message: "Deployment was successful.\n========================================"
                    )
                } catch(push) {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_FAIL_COLOR,
                        message: "Deployment failed.\n========================================"
                    )
                    throw push;
                }
            }
        }
    }
}
```
<br><br>

### Slack Notification

Jenkins와 Slack을 연동하여 파이프라인 상태 실시간 알림
<br>

**https://워크스페이스.slack.com/apps** 에 접속하여 **Jenkins Ci 앱** 설치<br>
Jenkins Ci 설정 지침 단계에 따라 구성

![alt text](img/image-1.png)

<aside class="warning">💡 플러그인 관리 → kubernetes, slack notification 설치</aside><br>

<aside class="warning">💡 시스템 설정 → GitHub Server, slack 연결</aside><br>

<aside class="warning">💡 Node 관리 → Clouds → New Cloud → WebSocket Check</aside><br><br>

---

## ArgoCD

GitOps 기반 지속적 배포 관리

- 자동 동기화 및 Slack 알림 연동
<br>

### ArgoCD Deploy
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f R-D/argocd/.
```
<br>

```
# 암호 찾기
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
<br>

### ArgoCD Notification

ArgoCD 상태 변경 시 Slack 알림
<br>

**Create ArgoCD App**

https://api.slack.com/apps

![alt text](img/image-5.png)
![alt text](img/image-6.png)
<br>

**OAuth & Permissions Bot Token Scopes Setting**

![alt text](img/image-7.png)
![alt text](img/image-8.png)
<br>

**argocd-notifications-secret.yaml**
```
apiVersion: v1
kind: Secret
metadata:
  labels:
    app.kubernetes.io/component: notifications-controller
    app.kubernetes.io/name: argocd-notifications-controller
    app.kubernetes.io/part-of: argocd
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  slack-token: <SLACK_TOKEN>
```
<br>

**Argocd Notifiaction CM Install**
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/notifications_catalog/install.yaml
```
<br>

**argocd-notifications-cm.yaml**
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
```
<br>

**argocd-notifications-controller.yaml**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: notifications-controller
    app.kubernetes.io/name: argocd-notifications-controller
    app.kubernetes.io/part-of: argocd
  name: argocd-notifications-controller
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-notifications-controller
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: argocd-notifications-controller
    spec:
      containers:
      - args:
        - /usr/local/bin/argocd-notifications
        env:
        - name: SLACK_TOKEN
          valueFrom:
            secretKeyRef:
              name: argocd-notifications-secret
              key: slack-token
        - name: ARGOCD_NOTIFICATIONS_CONTROLLER_LOGFORMAT
          valueFrom:
            configMapKeyRef:
              key: notificationscontroller.log.format
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_NOTIFICATIONS_CONTROLLER_LOGLEVEL
          valueFrom:
            configMapKeyRef:
              key: notificationscontroller.log.level
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_APPLICATION_NAMESPACES
          valueFrom:
            configMapKeyRef:
              key: application.namespaces
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_NOTIFICATION_CONTROLLER_SELF_SERVICE_NOTIFICATION_ENABLED
          valueFrom:
            configMapKeyRef:
              key: notificationscontroller.selfservice.enabled
              name: argocd-cmd-params-cm
              optional: true
        - name: ARGOCD_NOTIFICATION_CONTROLLER_REPO_SERVER_PLAINTEXT
          valueFrom:
            configMapKeyRef:
              key: notificationscontroller.repo.server.plaintext
              name: argocd-cmd-params-cm
              optional: true
        image: quay.io/argoproj/argocd:v2.14.2
        imagePullPolicy: Always
        livenessProbe:
          tcpSocket:
            port: 9001
        name: argocd-notifications-controller
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
        volumeMounts:
        - mountPath: /app/config/tls
          name: tls-certs
        - mountPath: /app/config/reposerver/tls
          name: argocd-repo-server-tls
        workingDir: /app
      nodeSelector:
        kubernetes.io/os: linux
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      serviceAccountName: argocd-notifications-controller
      volumes:
      - configMap:
          name: argocd-tls-certs-cm
        name: tls-certs
      - name: argocd-repo-server-tls
        secret:
          items:
          - key: tls.crt
            path: tls.crt
          - key: tls.key
            path: tls.key
          - key: ca.crt
            path: ca.crt
          optional: true
          secretName: argocd-repo-server-tls
```
<br>

```
kubectl edit application -n argocd <application name>
```
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscriptions: |
      - trigger: [on-created, on-deleted, on-deployed, on-scaling-replica-set, on-rollout-updated, on-rollout-step-completed, on-rollout-aborted, on-analysis-run-failed, on-analysis-run-error, on-health-degraded, on-sync-failed, on-sync-running, on-sync-status-unknown, on-sync-succeeded]
        destinations:
          - service: slack
            recipients: [devops]
```
<br>

**Slack**

![alt text](img/image-9.png)
<br><br>

---

## Prometheus

메트릭 기반 모니터링 및 알림 관리

- 이상 발생 시 Slack 알림
<br>

### Prometheus Stack
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
git clone https://github.com/prometheus-community/helm-charts.git
```
<br>

```
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --debug --timeout 10m -n monitoring
kubectl apply -f R-D/prometheus/.
```
<br>

**Uninstall Command**
```
helm uninstall prometheus
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheusagents.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd scrapeconfigs.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
```
<br>

### Alertmanager
**Helm Chart Value.yaml**
```
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      slack_api_url: '<WEBHOOK_URL>'
      http_config:
        follow_redirects: true
    inhibit_rules:
      - source_matchers:
          - 'severity = critical'
        target_matchers:
          - 'severity =~ warning|info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'severity = warning'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'alertname = InfoInhibitor'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
      - target_matchers:
          - 'alertname = InfoInhibitor'
    route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
      - receiver: 'slack-notifications'
        matchers:
          - alertname =~ "InfoInhibitor|Watchdog"
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#devops'
        api_url: '<WEBHOOK_URL>'
        username: 'alertmanager'
        title_link: 'http://alertmanager.k-tech.cloud/#/alerts?receiver=slack-notifications'
        text: "<!channel>\n\n*Alert Details:*\n- *Status*: {{ .Status }}\n- *Instance*: {{ .CommonLabels.instance }}\n- *Severity*: {{ .CommonLabels.severity }}\n- *Description*: {{ .CommonAnnotations.description }}\n- *Summary*: {{ .CommonAnnotations.summary }}\n\n*Triggered Labels:*\n{{ range .CommonLabels.SortedPairs }}- *{{ .Name }}*: {{ .Value }}\n{{ end }}\n\n*Alert Start Time*: {{ range .Alerts }}{{ .StartsAt }}{{ end }}\n\n{{ if .Alerts.Resolved }}*Alert Resolved Time*: {{ range .Alerts }}{{ .EndsAt }}{{ end }}{{ end }}"
        send_resolved: true
```
<br>

**Prometheus Stack Update**
```
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring -f helm-charts/charts/kube-prometheus-stack/values.yaml
kubectl apply -f R-D/alertmanager/.
```
<br>

**Slack Notification**<br>

![alt text](img/image-4.png)
<br><br>

## Grafana

---

메트릭 및 로그 시각화와 분석

- Prometheus 및 Loki 데이터 시각화
<br>

```
kubectl apply -f R-D/grafana/.
```
<br>

**Grafana admin password**
```
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
<br>

![alt text](img/image-10.png)
<br><br>

---

## Loki

중앙 집중식 로그 수집 및 저장, Grafana로 분석
<br>