# autoscale-comparison
노드단위 vs 팟 단위 오토스케일링 비교


## 1. 노드(NODE) 단위 오토스케일링
AWS 오토스케일링을 이용한다

### 1-1. VPC 및 서브넷 생성
- 1개 VPC
- 두개 AZ(가용영역)에 Public / Private Subnet 2개씩 설치
- 보안그룹 생성

### 1.2. 웹서버 인스턴스 시작
- EC2 생성
- 생성시 `userdata`에 웹서버 설치 스크립트 포함

### 1.3. 오토 스케일링
- 오토 스케일링용 AMI 생성

#### 시연클릭
[![aws](http://img.youtube.com/vi/72a9rQpjIR8/0.jpg)](http://www.youtube.com/watch?v=72a9rQpjIR8)


### 1.2. EC2 Userdata

```
#!/bin/bash -ex
yum -y update
yum -y install httpd php mysql php-mysql
chkconfig httpd on
/etc/init.d/httpd start
if [ ! -f /var/www/html/lab2-app.tar.gz ]; then
cd /var/www/html
wget https://us-west-2-aws-staging.s3.amazonaws.com/awsu-ilt/AWS-100-ESS/v4.0/lab-2-configure-website-datastore/scripts/lab2-app.tar.gz
tar xvfz lab2-app.tar.gz
chown apache:root /var/www/html/lab2-app/rds.conf.php
fi
```

## 2. 팟(POD) 단위 오토스케일링
Kubernetes Horizontal Pod Autoscaler를 이용한다

### 2-1. 환경설정
Google Cloud Platform을 이용한다.

```
gcloud config set compute/zone us-central1-a
gcloud container clusters create k8s-astest --num-nodes 5 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"  --enable-network-policy
```

생성된 노드 확인
```
[~]$ k get no
NAME                                        STATUS    ROLES     AGE       VERSION
gke-k8s-astest-default-pool-0847b08d-5dlw   Ready     <none>    2m        v1.9.7-gke.6
gke-k8s-astest-default-pool-0847b08d-fx5w   Ready     <none>    2m        v1.9.7-gke.6
gke-k8s-astest-default-pool-0847b08d-jjwv   Ready     <none>    2m        v1.9.7-gke.6
gke-k8s-astest-default-pool-0847b08d-tjwm   Ready     <none>    2m        v1.9.7-gke.6
gke-k8s-astest-default-pool-0847b08d-wvv6   Ready     <none>    2m        v1.9.7-gke.6
```

### 2-2. Pod Deploy & Service 생성 (ClusterIp)

```
$ kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80
service/php-apache created
deployment.apps/php-apache created
```

> 참고 : index.php 내에는 CPU 부하를 유발시키는 코드가 들어있다
```
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
```

### 2-3. Horizontal Pod Autoscaler 생성

오토스케일 정책을 세운다. 이하 명령어는 cpu 사용률이 50%을 넘어가는 경우 최소 1개 Pod으로 시작하여 최대 10개까지 증가시킨다.

```
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled
```

초기는 0%
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%  1         10        1          18s
```

### 2-4. 부하 발생
배포된 서비스를 연속으로 호출하여 부하를 발생시킨다.
```
$ kubectl run -i --tty load-generator --image=busybox /bin/sh

Hit enter for command prompt

$ while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```

CPU사용량이 증가하는 것을 볼 수 있다.
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET      CURRENT   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   305% / 50%  305%      1         10        1          3m
```

임계치인 50%를 초과하여 시간이 지날수록 Pod의 개수가 늘어난다.
```
$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   7         7         7            7           19m
```

### 2-5. 부하 발생 중단

busybox에서 도는 무한루프를 중단시키면, CPU의 부하가 감소하게 된다. 그리고 Pod의 개수는 줄어들어 다시 최종 한개만 남는다.

```
$ kubectl get hpa
NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%     1         10        1          11m

$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   1         1         1            1           27m
```

#### 시연클릭

[![k8s](http://img.youtube.com/vi/_4Xv9cufFFI/0.jpg)](http://www.youtube.com/watch?v=_4Xv9cufFFI)
