# Kubernetes
책 전반적으로 kubernetes에 선언적으로 의도하는 상태를 선언하고 kubernetes가 그것을 달성할수 있는 가장 좋은 방법을 찾으냄으로써 스스로 그 상태를 달성하도록 하는데 초점을 둔다

## Service
- 다이나믹하게 변하는 pod들에 대한 access를 추상화 하는 마치 pod과 같음
- selector를 이용해서 pod을 발견한다. kubernetes의 특성상 해당 라벨에 해당하는 pod이 현재 없을수도 있고(endpoints) 중간에 언제든 해당 라벨을 가진 pod이 추가되거나 사라질수도 있다
- externalService를 이용해서 클러스터 내부가 아닌 외부의 서비스를 추상화 할 수 있다
  - ex_) 외부 서비스 도메인 주소를 변경해도 서비스를 바라보는 다른 팟이나 서비스가 변경되지 않고 매끄럽게 변경이 가능
- kubernetes의 특성상 해당 서비스는 엔드포인트가 없을수도 있으나 해당하는 selector로 매칭된 팟이 생기면 서비스 된다. 즉 런타임에 해당하는 상태에 맞는 대상을  서비스한다

## ConfigMap
- 취지는 어플리케이션과 설정을 분리하는데 있다
- `k edit cm [config]`로 변경하면 바로는 아니지만 최대 1분내 configMap을 참고하고 있는 팟에 갱신이 된다. 문제는 마운트시 `subPath`로 일부 목록을 추가해서 쓰는 경우 edit을 해도 변경사항이 반영되지 않는다

## Secret
- configMap과 유사하지만 보안 정보가 섞여있을때 사용한다
- 기본적으로 `tmpfs`로 인메모리에만 기록된다. disk에 써지면 곤란하다고 판단. secret을 필요로 하는 해당 pod의 노드에만 기록된다. 마스터 노드(정확히는 etcd)가 가지고 있는데 1.7 버전 이후로는 마스터 노드에도 암호화 되서 기록된다. api server의 권한 관리도 하므로 함부로 가져갈 수 없다
- 생성시 도커 레지스트리용인지 configMap과 같은 목적인지에 따라 명령이 나뉜다
  - `$ k create secret docker-registry my-docker-hub-registry --docker-username=... --docker-email=... --docker-email=...`
  - `$ k create secret generic some-secret --from-file ./secret-files`

## deployment
- rs(replicaSet, replicationController의 대체)를 이용해서 팟을 관리하는 상위 레벨 리소스(팟을 직접 관리하지 않음)
- 팟을 관리하는 rs로 팟을 관리하게되면 예를들어 이미지 업데이트시 v1, v2를 롤링업데이트 할 때 v2 rs를 직접 따로 만들어 v2에서 팟을 하나 생성하고 v1에서 팟을 하나 죽이는 식으로 수동으로 처리를 해줘야 한다. 물론 `k rolling-update`라는 명령이 이를 해주긴하지만 해당 명령은 클라이언트가 API로 kubernetes에 직접 명령을 내리기 때문에 통신두절로 인해 중간상태로 남을수 있는 여지가 있다. 그리고 v1, v2 rs를 직접생성해줘야 한다고 했는데 v1,v2에 팟에 대해 라벨이 같아서 업데이트 후 v1 rs삭제시 라이브한 모든 팟까지 삭제할 위험도 있다. 이를 해결하기 위해 pod에 라벨을 추가해 각 rs에 속하는 팟을 구분해 줘야 한다. 이 모든 문제를 해결해 주는게 deployment다. `rolling-update`처럼 클라이언트 레벨에서 직접적인 명령을 내리는게 아니고 선언적으로 원하는 상태를 선언하면 kubernetes 컨트롤 플레인에서 선언한 상태로 맞춘다. `--v 6` 옵션을 통해 `rolling-update` 명령을 테스트 해보면 API 통신 목록을 일일이 확인할 수 있다. deployment는 팟에 해시를 두 단계를 둬서 팟과 rs를 알아서 구분한다. 기본 배포 전략인 rollingUpdate시 완료되서 이전 버전 rs에 replica가 0으로 되면 삭제하는 `rolling-update` 명령과는 달리 그 상태로 남아있는걸 확인할 수 있는데(실제로 rs 목록을 확인하면 2개의 셋이 있다) 롤백을 위해서다. `k rollout status deploy kubia`로 이전에 해당 deployment로 실행한 명령어를 확인할 수 있는데(`--record=true` 옵션을 줘야만 change-cause에 표시된다) `k rollout undo deploy kubia`로 이전 명령으로 롤백을 수행할 수 있다
  ```zsh
  k rollout history deploy kubia
  deployment.extensions/kubia
  REVISION  CHANGE-CAUSE
  1         kubectl create --filename=kubia-deployment-v1.yaml --record=true
  2         kubectl create --filename=kubia-deployment-v1.yaml --record=true
  3         kubectl create --filename=kubia-deployment-v1.yaml --record=true
  ```
  책의 내용과는 달리 내가 한건 `k edit deploy kubia`를 통해 image를 바꾸었는데 이렇게 표시가 된다. 책에서는 `k set image deploy kubia nodejs=luksa/kubia:v2`로 이미지 필드를 바꾸는데 그러면 change-cause에 이 명령줄에 해당 명령어로 나온다. 지금 정리하면서 보니 `--record`는 `k create -f kubia-deployment-v1.yaml --record`로 한 번만 수행을 하면 해당 리소스 관련해서 기록이 남는것 같다. 그 다음 명령어 부터는 옵션을 사용하지 않고 있다
- rollback은 `k rollout undo deploy kubia`로 한다. 그리고 진행 상황은 `k rollout status deploy kubia`로 본다. `--to-revision=1`로 
- rs는 revision 만큼 생성이 된다. 언제든 해당 revision으로 돌아갈 수 있다. 각 rs는 deployment의 버전별 스냅샷이다. 무한정 늘어날 수 없으니 `editionHistoryLimit` 속성으로 제한 가능하다. 기본값은 2이다.
  - `k explain deploy.spec` 으로 extentions/v1beta1 버전은 `revisionHistoryLimit`이 있고 int32 (i.e. 2147483647) 최대값이 디폴트이다
- `strategy.rollingUpdate`의 하위 속성으로 `maxSurge`, `maxUnavailable`이 있는데 배포중 몇 개의 팟이 허용 법위를 벗어나도 좋을지, 몇 퍼센트까지 replica에서 배포중 서비스 가능하지 않은 팟을 허용할지 설정할 수 있다
	- v1beta1 기준 기본값 1이 사용 
	- 백분율 변환시 반올림
	- 백분율 대신 절대값 사용 가능. ex) maxSurge: 1 이면 한 개의 파드가 더 추가 될 수 있음
- `minReadySeconds`는 롤아웃중 파드가 투입되고 다음 파드를 생성하기 전에 대기하는 시간으로 롤아웃 시간을 지연시킬수 있다. 실세계에서는 다음 파드를 투입하기 전에 생성한 파드가 트래픽을 받으면서 충분히 살 수 있는지 상태를 파악하기 위해 10초보다 훨씬 높은 시간으로 설정한다고 한다. 이런 지연 시간은 생성된 파드를 충분히 지켜볼 시간뿐만 아니라 잘못된 버전이 빠르게 나가버리는것을 막는 에어백과도 같은 효과를 지닌다.
  = `maxUnavailable`이 설정되면 `minReadySeconds`동안 대기하는 팟들은 준비된 상태가 아니기 때문에 다음 롤아웃 프로세스가 진행되지 않는다
	- 해당 시간동안 팟이 실패하면 롤아웃 프로세스 중단 -> 적절한readiness probe 설정과 더불어 잘못된 버전이 나가는걸 막는 효과
- 롤아웃이 오류로 인해 중단되면 `progressDeadlineSeconds`에 지정된 시간이 초과하면 다음과 같은 에러가 발생한다
  -'error: deployment "kubia" exceeded its progress deadline'
	- 에러 발생시 롤아웃 `undo`, `status` 명령시 위와 같은 오류로 더 이상 진행이 안된다
	- 에러 발생시 리소스를 업데이트 하면 다시 정상화 되는것을 확인했다(`k apply -f`)

## Statefulset
- 기존의 stateless pod을 관리하는 replicaSet과는 달리 이전과 동일한 아이덴티티를 갖는다. 상태(스토리지 볼륨), ~~동일한 ip(Headless service로)~~, 이름(stateless pod이 매번 다른 해시가 뒤에 붙는것과 달리 서수 인덱스. `name-0`), 호스트 이름 등을 갖는다
- stateless pod을 가축이라고 하면 stateful pod은 애완동물이라고 할 수 있다. 언제든 대체되어도 구별이 어려운 가축의 특성과 달리 키우는 애완동물은 대체가 되지 않는다
### Headless service

### Pod 마다 개별 볼륨

