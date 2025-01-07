# CI/CD와 PLG 실습 과정

## Azure Login

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

**Helm Install**
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Cert-manager
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

## Harbor
```
kubectl create ns devops-tools
kubectl apply -f RnR/harbor/harbor-certificate.yaml
kubectl get nodes --show-labels | grep linux
```

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

**Cluster Issuer**
```
kubectl apply -f cert-manager/.
```

## Istio

**Isito Download**
```
curl -L https://istio.io/downloadIstio | sh -
mv istio-1.24.1 istio
cd istio
export PATH=$PWD/bin:$PATH
cd ..
```

**Istio Deploy**
```
istioctl install --set profile=default -y
# istioctl install --set profile=demo --skip-confirmation
```

**auto sidecar injection**
```
kubectl label namespace default istio-injection=enabled --overwrite
kubectl label namespace istio-system istio-injection=enabled --overwrite
kubectl label namespace monitoring istio-injection=enabled --overwrite

kubectl get namespace -L istio-injection
```

**sidecar check**
```
istioctl experimental check-inject <pod-name>
```

## Jenkins
**Workflow**
![alt text](image.png)

**Deploy**
```
kubectl apply -f RnR/jenkins/.
```

**Password**
```
kubectl exec -it svc/jenkins-service -n devops-tools -- cat /var/jenkins_home/secrets/initialAdminPassword
```

### Slack Notification

**https://워크스페이스.slack.com/apps** 에 접속하여 **Jenkins Ci 앱** 설치
Jenkins Ci 설정 지침 단계에 따라 구성

![alt text](image-1.png)

<aside>
💡 **플러그인 관리 → kubernetes, slack notification 설치**
</aside>

<aside>
💡 **시스템 설정 → GitHub Server, slack 연결**
</aside>

<aside>
💡 **Node 관리 → Clouds → New Cloud → WebSocket Check**
</aside>

### Kaniko
**Docker** : Docker는 Docker 데몬이 호스트 시스템에서 실행되고 이미지를 빌드하는 데몬 기반 접근 방식을 사용합니다. 이를 위해서는 특히 Kubernetes 클러스터에서 보안 문제가 될 수 있는 권한 있는 액세스가 필요합니다.

**Kaniko** : Kaniko는 컨테이너 또는 Kubernetes 클러스터 내부의 Dockerfile에서 컨테이너 이미지를 빌드하는 도구입니다. 특별한 권한이 필요하지 않으므로 Kubernetes 환경의 보안이 더욱 강화됩니다.

**Create secret**
```
docker login https://harbor.k-tech.cloud
cat ~/.docker/config.json
cat ~/.docker/config.json | base64
```

**regcred.yaml**
```
apiVersion: v1
kind: Secret
metadata:
  name: docker-config-secret
  namespace: devops-tools
data:
  .dockerconfigjson: 인코딩한 데이터
type: kubernetes.io/dockerconfigjson
```

```
kubectl apply -f RnR/kaniko/.
```

### Pipeline
**Configuration**
<aside>
💡 check : **Do not allow the pipeline to resume if the controller restarts**
</aside>

<aside>
💡 check : **GitHub hook trigger for GITScm polling**
</aside>

**Json**
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
```