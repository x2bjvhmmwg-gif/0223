

# README.md (총정리)

# 2026-02-23 실습 총정리: Docker, SSH, AWS EC2, Kubernetes

오늘 실습에서는 **SSH 설정, Docker 컨테이너, AWS EC2, Kubernetes Pod 관리**까지 실습하였고, 발생한 오류 및 원인도 포함하여 정리합니다.  



## 1️⃣ SSH 키 생성 (Windows)

- 명령어:
```cmd
ssh-keygen -t ed25519
````

* 생성되는 파일:

  * `id_ed25519` : 개인 키
  * `id_ed25519.pub` : 공개 키
* 오류 및 주의사항:

  * Windows CMD에서는 `mkdir ~/.ssh` 대신 `mkdir %USERPROFILE%\.ssh` 사용
  * `cp` 명령어는 Linux 명령어이므로 Windows CMD에서 작동하지 않음 → `copy` 사용
* 공식 문서: [OpenSSH Keygen](https://www.openssh.com/manual.html)

---

## 2️⃣ Docker에서 SSH 설정

### Dockerfile 예시 (Ubuntu + SSH)

```dockerfile
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install -y openssh-server sudo
RUN mkdir /var/run/sshd

RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config
RUN echo "root:1234" | chpasswd

RUN useradd -m -s /bin/bash study
RUN echo "study ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

RUN mkdir /home/study/.ssh && chmod 700 /home/study/.ssh
COPY authorized_keys /home/study/.ssh/authorized_keys
RUN chown -R study:study /home/study/.ssh

EXPOSE 22
CMD ["/usr/sbin/sshd","-D"]
```

* 실행:

```bash
docker build -t ubuntu-ssh .
docker run -d -p 22:22 --name os ubuntu-ssh
ssh study@localhost
```

* 오류 및 해결:

  * `docker cp` 사용 시 `/home/study/.ssh` 폴더가 없으면 에러 발생

    * 해결: `kubectl exec`로 접속 후 `mkdir ~/.ssh` 및 `chmod 700 ~/.ssh` 실행
  * 권한 문제:

    * `authorized_keys` 파일 소유자가 root이면 `Operation not permitted`
    * 해결: root로 접속 후 `chown study:study /home/study/.ssh/authorized_keys`

* 공식 문서: [Docker SSH in Container](https://docs.docker.com/engine/examples/running_ssh_service/)

---

## 3️⃣ AWS EC2 접속

* EC2 생성:

  * AMI: Ubuntu 24.04
  * 프리티어 인스턴스
  * 키 페어 생성 `.pem`
  * 보안 그룹: 22, 80 오픈
* 접속:

```bash
ssh -i aws-key.pem ubuntu@<EC2-DNS>
```

* Nginx 설치:

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl status nginx
```

* 공식 문서:

  * [EC2 Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)
  * [Nginx 설치](https://nginx.org/en/docs/install.html)

---

## 4️⃣ Docker 설치 및 권한 설정 (EC2)

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

* 오류:

  * `docker: permission denied` → 해결: 현재 사용자 docker 그룹 추가 후 로그아웃/재접속
* Docker Compose 설치:

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
-o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

* 공식 문서:

  * [Docker Compose 설치](https://docs.docker.com/compose/install/)

---

## 5️⃣ Kubernetes 실습

### Pod 정의 예시

#### Nginx Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web2
spec:
  containers:
  - image: nginx:1.28
    name: web
    ports:
    - containerPort: 80
```

#### FastAPI Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fastapi
spec:
  containers:
  - image: backend
    name: fastapi
    ports:
    - containerPort: 8000
    imagePullPolicy: IfNotPresent
```

#### MariaDB Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: maria
spec:
  containers:
  - image: mariadb:12.1.2
    name: maria
    ports:
    - containerPort: 3306
    env:
      - name: MARIADB_ROOT_PASSWORD
        value: "1234"
      - name: MARIADB_DATABASE
        value: edu
```

### 명령어

```bash
kubectl apply -f <pod.yaml>
kubectl get pods -o wide
kubectl exec -it <pod_name> -- /bin/bash
kubectl port-forward pod/<pod_name> <local_port>:<container_port>
kubectl delete all --all
```

* 오류 및 원인:

  * `ErrImagePull`: 이미지가 로컬/레지스트리에 없거나 Pull 권한 없음
  * `curl` 명령어 없음: Pod 내부 Ubuntu 컨테이너에 curl 설치 필요 (`apt install -y curl`)
  * 포트 충돌: 같은 호스트 포트는 하나만 가능 → `kubectl port-forward`로 포트 조정

* 공식 문서:

  * [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
  * [kubectl port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application/)
  * [kubectl exec](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#exec)

---

## 6️⃣ 오늘 실습 총 흐름

1. Windows에서 SSH 키 생성 → `authorized_keys` 준비
2. Docker 컨테이너에서 SSH 서비스 구축 → 권한 설정, 비밀번호 없이 접속
3. AWS EC2 생성 후 접속 → Nginx 설치 → Docker 설치
4. Docker 이미지 빌드 및 실행 (FastAPI, Ubuntu SSH)
5. Kubernetes 설치/사용 → Pod 생성 → 내부 컨테이너 접속 → 포트 포워딩 테스트
6. 오류 발생 시 원인 확인 → 공식 문서 참조 → 해결

---

## 7️⃣ 참고 링크

* [OpenSSH](https://www.openssh.com/manual.html)
* [Docker Official Docs](https://docs.docker.com/)
* [AWS EC2 Linux Access](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)
* [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
* [kubectl Commands](https://kubernetes.io/docs/reference/kubectl/)

---

**Tip:**

* Pod 이름은 중복 불가, 컨테이너 이름은 동일 가능
* `docker cp` 또는 `kubectl exec`로 권한 설정 필수 (`chmod`, `chown`)
* 포트포워딩 시 호스트→Kubernetes→Pod→컨테이너 순서로 연결됨

```

