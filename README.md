# Kubernetes

## Service
- 다이나믹하게 변하는 pod들에 대한 access를 추상화 하는 마치 pod과 같음
- selector를 이용해서 pod을 발견한다
- externalService를 이용해서 클러스터 내부가 아닌 외부의 서비스를 추상화 할 수 있다
  - ex_) 외부 서비스 도메인 주소를 변경해도 서비스를 바라보는 다른 팟이나 서비스가 변경되지 않고 매끄럽게 변경이 가능

## ConfigMap
- 취지는 어플리케이션과 설정을 분리하는데 있다
- `k edit cm [config]`로 변경하면 바로는 아니지만 최대 1분내 configMap을 참고하고 있는 팟에 갱신이 된다. 문제는 마운트시 `subPath`로 일부 목록을 추가해서 쓰는 경우 edit을 해도 변경사항이 반영되지 않는다

## Secret
- configMap과 유사하지만 보안 정보가 섞여있을때 사용한다
- 기본적으로 `tmpfs`로 인메모리에만 기록된다. disk에 써지면 곤란하다고 판단. secret을 필요로 하는 해당 pod의 노드에만 기록된다. 마스터 노드(정확히는 etcd)가 가지고 있는데 1.7 버전 이후로는 마스터 노드에도 암호화 되서 기록된다. api server의 권한 관리도 하므로 함부로 가져갈 수 없다.
- 생성시 도커 레지스트리용인지 configMap과 같은 목적인지에 따라 명령이 나뉜다
  - `$ k create secret docker-registry my-docker-hub-registry --docker-username=... --docker-email=... --docker-email=...`
  - `$ k create secret generic some-secret --from-file ./secret-files`
