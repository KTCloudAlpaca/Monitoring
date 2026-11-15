# Monitoring
install monitroing systems and control the services

– Monitoring –

install? default “master”입니다.

Structure?

|서비스 이름|기능|비유|
|--------| ----|---|

Prometheus: 수집, 저장, 조회 
-> 건물의 온도, 전력, 장비 상태를 주기적으로 측정하는 센서, 측정한 값을 기록하는 현황판

Grafana: 그래프, 대시보드
-> 건물 관제실

Alertmanager: 알림
-> 건물 누수 발생 시 관리자 폰에 알람

Operator: monitoring 시스템을 k8s안에서 생성하고 관리
-> 건물 관리자


Why? 원스톱 패키지 배포(모니터링 환경을 수동 설정 없이 하나로 빠르고 표준화되게 구축/운영 가능)
-> toss vs 예전 타 은행 앱

0) namespace 설정
—--------------------------------------
kubectl create namespace monitor
kubectl get namespace monitor
—--------------------------------------


1)  helm 활용
—--------------------------------------
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
helm search repo prometheus-community

helm install mymonitor \
  prometheus-community/kube-prometheus-stack \
  --version 92.3.0 \
  --namespace monitor \
  --wait

#설치 확인
helm list -n monitor
—--------------------------------------


2) service, pod/static nodeport 설정
—--------------------------------------
kubectl get svc,pod,deploy,sts -n monitor -o wide

#ClusterIP -> static NodePort 변경 
kubectl edit svc mymonitor-kube-prometheus-prometheus -n monitor
kubectl edit svc mymonitor-grafana -n monitor
kubectl get svc,pod -n monitor -o wide
—--------------------------------------


3) nfs 설정
—--------------------------------------
sudo dnf update -y
sudo dnf install -y nfs-utils\
rpm -q nfs-utils

#nfs 활성화 및 상태 확인
sudo systemctl enable --now nfs-server.
sudo systemctl status nfs-server --no-pager
-> 위 두 단계를 worker에서 동일하게 진행

#모니터링 데이터(log, tsdb)를 저장할 디렉터리 생성
sudo mkdir -p /data/prometheus-log
sudo mkdir -p /data/prometheus-tsdb

#권한 변경
sudo chmod 755 /data/prometheus-log
sudo chmod 755 /data/prometheus-tsdb

#nfs 공유 설정
sudo vi /etc/exports
/data/ 10.120.0.140(rw,sync,no_root_squash,no_subtree_check) 10.120.0.141(rw,sync,no_root_squash,no_subtree_check)

#설정 적용
sudo exportfs -rav
sudo exportfs -v

#워커에서 확인
showmount -e 10.120.0.140

sudo mount -v -t nfs \
  -o vers=4.1 \
  10.120.0.140:/data \
  /mnt/nfs-test

findmnt /mnt/nfs-test
df -hT /mnt/nfs-test
—--------------------------------------


4) storageclass 활용해 동적 volume 생성
—--------------------------------------
#provisioner(volume을 할당 기능) 대체 csi 설치
helm repo add csi-driver-nfs \
  https://kubernetes-csi.github.io/csi-driver-nfs

helm repo update
helm search repo csi-driver-nfs/csi-driver-nfs --versions

helm install csi-driver-nfs \
  csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --version 4.13.4

#설치 확인
helm status csi-driver-nfs -n kube-system
kubectl get pods -n kube-system -o wide
kubectl get csidriver nfs.csi.k8s.io

#nfs csi용 storageclass 생성
vi prometheus-tsdb-storageclass.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prometheus-tsdb
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.120.0.140
  share: /data/prometheus-tsdb
  onDelete: retain
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - nfsvers=4.1

kubectl apply -f prometheus-tsdb-storageclass.yamll
kubectl get storageclass prometheus-tsdb -o yaml

#nfs csi용 pvc 생성
vi prometheus-tsdb-pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-tsdb-pvc
  namespace: monitor
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: prometheus-tsdb
  resources:
    requests:
      storage: 5Gi

kubectl apply -f prometheus-tsdb-pvc.yaml
kubectl get pvc -n monitor
kubectl get pv


#생성한 volume과 prometheus 메트릭 데이터 연결 
vi prometheus-tsdb-values.yaml

prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: prometheus-tsdb
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi

helm upgrade mymonitor prometheus-community/kube-prometheus-stack \
  --namespace monitor \
  --version 92.3.0 \
  --reuse-values \
  -f prometheus-tsdb-values.yaml 
—--------------------------------------

5) 앱과 프로메테우스 monitoring 연결 
—--------------------------------------
#에플리케이션 메트릭 노출
#javaScript 기준
vi ~/Dev/backend/src/config/metric.js

const client = require('prom-client');
const express = require('express');
const app = express();

#기본 메트릭 수집 활성화
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({ timeout: 5000 });

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(3000);


#k8s 환경에서 에플리케이션의 metric을 주기적으로 긁어오도록 설정
vi myapp-servicemonitor.yaml

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitor
  labels:
    release: mymonitor
spec:
  selector:
    matchLabels:
      app: backend 
  endpoints:
  - port: http 
    path: /metrics
    interval: 15s

kubectl apply myapp-servicemonitor.yaml -n monitor
—--------------------------------------

6) Grafana dashboard 접근 및 활용 
—--------------------------------------
#hostPC(window) 관리자 터미널에서 원격 온프레미스 접근(prometheus)
#Window PowerShell
PS C:\Users\user> ssh -L 3004:127.0.0.1:3004 -p 22140 student@43.202.217.158 "kubectl port-forward -n monitor svc/mymonitor-kube-prometheus-prometheus 3004:9090"

#chrome 통해 접근
http//localhost:3004

status > target에 연결한 backend가 잘 등록 되어 있는지 확인


#hostPC(window) 관리자 터미널에서 원격 온프레미스 접근(grafana)
#Window PowerShell
PS C:\Users\user> ssh -L 3003:127.0.0.1:3003 -p 22140 student@43.202.217.158 "kubectl port-forward -n monitor svc/mymonitor-grafana 3003:80"

#linux shell master
helm status mymonitor -n monitor

#password 확인: 빨간색 상자 부분 명령어 그대로 입력

#chrome 통해 접근
http//localhost:3003


#그라파나 대시보드 생성 및 연동
1.새 대시보드 생성:1단계.
그라파나 좌측 메뉴에서 Dashboards > New > New dashboard를 클릭
2.패널(Panel) 추가 및 쿼리 작성:2단계.
Add visualization을 누르고 데이터소스로 Prometheus를 선택
쿼리(Metric 입력창)에 앱에서 수집 중인 메트릭 키를 입력
3.시각화 및 저장
우측 패널 설정에서 그래프 종류(Time series, Stat 등)를 선택하고 이름을 지정
우측 상단의 Save 버튼을 눌러 대시보드를 저장
—--------------------------------------







What?
pvc:


pv:


storageclass:


1. pv/pvc 활용 -> 정적 volume 생성

 
2. storageclass 활용 -> 동적 volume 생성

따로 디스크 할당 없이 알아서 pvc에 맞는 pv를 생성해 volume 생성


Why? 왜 구현 했을까?

Problem? 로컬 환경에서는 자동으로 디스크를 할당해주는 기능을 담당하는 ‘provisioner’ 기능이 X 

Solution? PVC 요청에 따라 NFS 공유 아래에 하위 디렉터리를 생성해 동적 PV를 제공하는 (NFS) CSI 드라이버 활용

helm install csi-driver-nfs \
  csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --version 4.13.4

kubectl get pods -n kube-system -o wide | grep csi-nfs
kubectl get csidriver nfs.csi.k8s.io

 


