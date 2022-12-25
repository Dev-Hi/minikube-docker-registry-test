# Docker registry in Minikube

minikube에서 docker registry를 구축하고, spring app을 빌드한 image를 docker registry을 push하여, 이 image를 minikube에 deploy 합니다.

<br/>

## 개발 환경

Minikube 구축 및 spring app 개발은 wsl2 위에서 진행됐습니다.

- minikube v1.28.0

- Window10

- wsl2
  
  - ubuntu 22.04.1 LTS

- kubectl v1.25.3

- spring boot 2.7.3

- jdk8

- k9s

<br/>

다시 한번 언급하지만 모든 진행 과정은 **wsl2**에서 진행 됐습니다.

<br/>

## Docker registry 구축

아래의 명령어를 입력하여 registry addon 활성화합니다.

```shell
minikube addons enable registry
```

<br/>

여기서 docker registry가 사용하는 포트번호가 5000이 아닌, 다른 포트번호를 사용하라는 출력이 나왔다면 그 포트번호를 사용해야합니다. 이 경우, 아래의 문구처럼 나옵니다. <br/>

> Registry addon with docker driver uses port 57452 please use that instead of default port 5000 

<br/>

여기서는 57452가 docker registry가 사용하는 포트번호로 나왔기 때문에, 이를 예제로 사용하겠습니다. <br/>

client가 http를 통해 image를 pull할 수 있도록 설정합니다. 기본적으로 service cluster ip는 `10.0.0.1`에서 사용가능하기 때문에, 이 범주에 있는 ip는 http를 사용할 수 있도록 설정합니다. (클러스터 ip는 임의로 설정되기 때문입니다.)

```shell
minikube config set insecure-registry "10.0.0.0/24"
```

<br/>

docker registry가 minikube에 정상적으로 구축됐다면, 다음의 명령어를 입력해 docker registry가 갖고 있는 image들을 확인합니다.(push된 image가 없기 때문에 아무것도 나오지 않을 것입니다.)

```shell
curl http://localhost:57452/v2/_catalog  
```
<br/>
출력의 결과는 아래처럼 나올 것입니다.

```json
{"repositories":[]}
```

<br/>

간단한 spring app을 만듭니다. 간단한 spring app이기 때문에, WSL2에서 spring app 개발 테스트를 사용했던 프로젝트를 사용했습니다.

<br/>

## Spring app build, image build, pushing image to registry

spring app build, image build, pushing image to docker registry 과정까지 쉽게 하기 위해 `jib`를 사용합니다. 따라서 `build.gradle`에 아래의 내용을 추가합니다.

```groovy
plugins {
    id 'com.google.cloud.tools.jib' version '3.3.1'
}

jib {
    to {
        image = "localhost:57452/example"
    }

    // local docker registry를 사용하기 때문에, http 허용
    allowInsecureRegistries true
}
```

<br/>

`./gradlew jib`를 입력한 후, `curl http://localhost:57452/v2/_catalog` 입력하면, image가 registry에 반영된 것을 확인할 수 있습니다.

```json
{"repositories":["example"]}
```

마지막으로 할 일은 이제 app에 대한 deployment, service 작성 및 실행 결과를 확인하는 것입니다.

<br/>

## Deploy spring app to minikube

하나의 `yaml` 파일에서 `deployment`, `service`를 작성하여, 한 번의 명령어로 이 2개를 minikube에 반영할 것입니다. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: yourClusterIP/example
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31212
```
<br/>
위의 파일에서 `yourClusterIP/example`이 존재하는 것을 확인할 수 있는데, 여기서 docker registry cluster에대한 ip를 입력합니다. 아래의 명령어를 입력하여, docker registry cluster ip를 확인합니다.

```shell
kubectl get services --namespace kube-system
```
<br/>
`NAME`이 registry를 갖는 행에서 `CLUSTIER-IP`정보가 있을 것입니다. 이 정보를 입력합니다. 마지막으로 아래의 명령어를 입력하여 `deployment`, `service`를 minikube에 반영합니다. 

```shell
kubectl apply -f example-app-deployment.yaml
```
<br/>
위의 명령어 입력하실 때, 파일이 있는 경로에 실행해주면 됩니다. 참고로 k9s를 통해 확인하면, 배포가 정상적으로 됐는지 쉽게 확인할 수 있습니다. 

<br/>

## Test

정상적으로 됐는지 확인합니다. 테스트를 위해, window 10에 있는 chrome web browser를 사용했습니다. minikube에 있는 spring app에 대한 `service`를 통해 접근할 것이기 때문에, 아래의 명령어를 입력합니다.

```shell
minikube service myapp-svc
```

<br/>

마지막 3줄에 아래와 같은 내용이 나오는 것을 볼 수 있는데, <br/>

🎉  Opening service default/myapp-svc in default browser... <br/>
👉  `http://127.0.0.1:39185` <br/>
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it. <br/>

<br/>

위에서 주어진 url를 이용하여 `http://127.0.0.1:39185/ping`을 chrome에 입력합니다.

그러면 `pong`이 나옵니다.
