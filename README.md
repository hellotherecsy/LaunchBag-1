# TCL LaunchBag User Guide
- Launch(Lunch)와 Bag의 합성어로 언제/어디서나 실행가능한 개발/분석/운영 환경이 가능한 서비스
  - Launch : 실행하다 (Lunch와 비슷한 발음)
  - Bag : 가방 (가방처럼 어디든지 이동 가능, 클라우드 환경 또는 로컬 서버환경 등)

## [STEP 1]. Install LaunchBag package using ansible-playbook
- Ansible을 이용한 Docker설치 및 관련 라이브러리 설정
  - Ansible : 2.3.0
- Docker / Rancher Server 설치 및 실행
  - Rancher : 1.6.10
  - Docker : 17.09.0-ce
- Rancher Agent 설치 및 실행

```
> ansible-playbook -i inventories/hosts rancher.yml
```

### LaunchBag Service Architecture
![service architecture](https://github.com/freepsw/LaunchBag/blob/master/images/launchbag-install.png)
- 사용자의 click 한번으로 필요한 모든 설정을 자동화하고 배포


### 01. 웹서버 접속
- Rancher 서버 접속 : http://rancher-server-ip:8080
- LaunchBag 웹서버 접속 : http://launchbag-ip:8090

### 02. Rancher Server 접속 : Rancher 인증키 발급
- 발급받은 Key 정보를 LaunchBag 설정정보에 입력
- 해당 Key를 이용하여 rancher를 통한 container 서비스를 구동
- http://rancher-server-ip:8080/env/1a5/api/keys
> Access Key (UserName) : A843B36E7449418AB52D
> Secret Key (Password) : oExt8NR6DtCQUawmz3P58TrhDfX6yDMYPT1krA9H


## [STEP 2] Run LaunchBag for data scientist
- 분석을 위해 필요한 시스템 설정(system library 및 R/python packages 등)이 완료된 환경 제공
- 모든 분석환경이 구셩된 jupyter notebook(python3 + R)을 빠르게 생성하여, 지연없이 분석환경 구성 가능
- 분석가는 해당 URL에 접근하여 분석업무 수행!

### LaunchBag 서비스 배포 구성
![service architecture](https://github.com/freepsw/LaunchBag/blob/master/images/launchbag-service.png)
- ansible-playbook script로 서비스 배포에 필요한 설정 및 api 접속 완료
- 실제 서비스는 Rancher를 이용하여 구동 (서비스 안정성 및 확장을 위함)

### 01. Connect to LaunchBag Web page
- LaunhBag 서버에 접속하여, 어떤 서비스를 실행 할 것인지 선택한다.
- 여기서는 Data scientist를 위한 LaunchBag을 실행하므로,
- 필요한 설정값을 입력하
- http://launchbag-ip:8090

#### LaunchBag 설정 값
- Default는 아무 설정이 필요없다. (실행시 R, Python이 kernel이 설치된 Jupyter Notebook 환경 제공)
- 사용자가 추가로 설정할 수 있는 정보
  - [분석 파일] : ipynd, .py, .r 등 분석언어별 실행파일
  - [데이터 파일] : 테스트에 필요한 데이터 파일
- 시스템이 자동으로 설정하는 정보
  - [서비스 명]
    * container로 구동될 서비스 고유 명칭 (개인별로 고유한 값, 예를 들어 launchbag-datascience)
    * 중복되지 않도록 시스템에서 "시간+Seq"로 일련번호 생성 필요
  - [Jupyter Port]
    * 하나의 서버에 여러개의 jupyter notebook이 실행될 수 있으므로,
    * Jupyter port를 서로 다르게 지정해야 함 (Web에서 자동으로 생성해야 함)

### 02. Run LaunchBag
- Web 화면에서 "Launch" 버튼을 클릭하면,
- 서비스가 배포되고, 잠시 후에 접속할 수 있는 Jupyter Notebook URL이 표시된다.
- 이제부터 분석을 시작할 수 있다. (약 10초 이내)

#### 내부 명령어 (LaunchBag Web Server에서 실행하는 명령)
```
> ansible-playbook -i inventories/hosts launchbag_datascience.yml -e "container_name=launchbag-datascience2"

# ansible 내부적으로는 아래의 명령어를 Rancher API로 변환하여 호출한다.
> docker run -it -p 8888:8888 -e GRANT_SUDO=yes  --user root  --name launchbag-datascience  freepsw/launchbag-juypter-r:v0.1.1  start-notebook.sh --NotebookApp.token=''

# RancherAPI로 변환하여 호출하는 방식
> curl -u "${CATTLE_ACCESS_KEY}:${CATTLE_SECRET_KEY}" \
-X POST \
-H 'Accept: application/json' \
-H 'Content-Type: application/json' \
-d '{"name":"LaunchBag-jupyter", "system":false, "dockerCompose":"{ \"version\": \"2\", \"services\": { \"launchbag-datascience\": { \"image\": \"freepsw/launchbag-juypter-r:v0.1.1\", \"stdin_open\": true, \"tty\": true, \"ports\": [ \"8888:8888/tcp\" ], \"command\": [ \"start-notebook.sh\", \"--NotebookApp.token=\" ], \"labels\": { \"io.rancher.container.pull_image\": \"always\" } } } }", "rancherCompose":"{ \"version\": \"2\", \"services\": { \"launchbag-datascience\": { \"scale\": 1, \"start_on_create\": true } } }", "startOnCreate":true, "binding":null}' \
'http://<ip>:8080/v2-beta/projects/1a5/stacks'
```

## 03. Jupyter notebook 접속 및 분석업무 시작!
- LaunchBag UI에서 사용자가 클릭할 수 있는 URL Link 제공 : http://jupyter-server-ip:8888
- token 없이 접속할 수 있도록 설정


## [ ETC ]
### 01.  R package 설치오류 해결
#### [Problem - package 설치시 library 오류]
- jupyter notebook에서 install.packages("igraph"), install.packages("randomcoloR") 실행시 오류
- 오류 메세지
```
install.packages("igraph")
 -> error: expected ‘)’ before ‘GRAPHML_NAMESPACE_URI’

install.packages("randomcoloR")
 -> ERROR: configuration failed for package ‘V8’
```
#### [Cause]
- package 설치에 필요한 library가 시스템에 설치되지 않았음.

#### [Solve 1 안]
- docker run 이후에,
- docker exec로 container 내부에 접속하여 필요한 library 설치 (apt-get)
- 이 방식은 매번 ansible로 관련 명령어를 실행해야 한다.
  - 장점 : 유연하게 다양한 library를 팰ㅇㅎ설치 가능
  - 단점 : docker run으로 실행된 후에 추가적인 library를 설치해야 하므로, 시간이 소요됨. (약간..)
```
> docker build -t launchbog-juypter-r .
> docker run -it --rm -p 8888:8888 -e GRANT_SUDO=yes  --user root jupyter/r-notebook start-notebook.sh --NotebookApp.token=''

> docker exec -it 440188af9db7  sudo apt-get install -y libxml2-dev libxml2
> docker exec -it 440188af9db7  sudo apt-get install -y libv8-dev
```
- conda에서 R package 설치시 package 모듈 검색 (설치 방법)
- https://anaconda.org/conda-forge/randomcolor
```
conda install -c <your anaconda.org username> custom-r-bundle
```


#### [Solve 2 안]
- Dockerfile을 시스템에 필요한 library(libxml2-dev libxml2 libv8-dev) 및 conda library(r-igraph') 설치 명령어 추가 --> docker build로 이미지 생성
- 이 방식으로 해도, conda install 로 설치가 안되는 R package가 있다. (내가 방법을 못찾은 걸수도 있음)
##### - [ 해결 ] docker commit 활용
  - 이런 경우에는 docker run 이후,
  - jupyter notebook에서 필요한 r package(conda install로 설치 안되는 것, components, randomcoloR) 설치
  - 설치가 완료된 후에 동작중인 docker를 commit으로 image로 생성 (R package가 설치된 이미지)
```
> docker commit  761863a2c280<실행중인 container id>  freepsw/launchbag-juypter-r:v0.1.1 <새로 생성할 image name>
```




### 02. Install.packages("randomcoloR")설치 시 UnsatisfiableError 에러
- conda install로 설치시 아래와 같이 python version과 관련된 문제가 발생한다
#### [ Error ]
```
UnsatisfiableError: The following specifications were found to be in conflict:
  - python 3.6*
  - randomcolor -> python 2.7*
Use "conda info <package>" to see the dependencies for each package.
```
#### [Solve]
  - 동작중인 container 에 접속하여 아래 명령어 실행 (해당 package가 python3.6 버전을 지원하지 못함)
  - conda install python=3.5


### 03. Register launchbag docker image to Docker-Hub

- 어디서나 내가 만든 이미지를 dockerhub에서 다운 받을 수 있도록 등록
```
> docker login --username=yourhubusername --email=youremail@company.com
> docker push  freepsw/launchbag-juypter-r:v0.1.1
```

### 04. SSH Connection without Authentication (ssh 초기 연결시 인증 체크 하지 않는 법)
- Are you sure you want to continue connecting (yes/no)? --> 이런 메세지 없도록
```
> export ANSIBLE_HOST_KEY_CHECKING=False
> ansible-playbook -i inventories/hosts ping.yml
```


### 05. Ansible uri modul의 body에 json 설정시, Rancher Server에 정상적으로 전송되지 않는 문제.
- 아래와 같은 ansible 스크립트 사용.
- 여기서 json은 별도의 파일에서 읽어옴. (json 문법 검증시 오류 없음을 확인, https://jsonformatter.curiousconcept.com/)
```
- name: Add rancher stack
  uri:
    user: "------"
    password: "------------"
    headers:
      Content-Type: "application/json"
    url: http://{{ rancher_server }}:8080/v2-beta/projects/1a5/stacks
    method: POST
    body: "{{ lookup('file','templates/launchbag-datascience.json') }}"
    force_basic_auth: yes
    body_format: json
    status_code: 201 # Created
```

- json 파일의 구성
```json
  {
     "name": "LaunchBag-jupyter",
     "system": false,
     "dockerCompose": {
        "version": "2",
        "services": {
           "launchbag-datascience2": {
              "image": "freepsw/launchbag-juypter-r:v0.1.1",
              "stdin_open": true,
              "tty": true,
              "ports": [
                 "8888:8888/tcp"
              ],
              "command": [
                 "start-notebook.sh",
                 "--NotebookApp.token="
              ],
              "labels": {
                 "io.rancher.container.pull_image": "always"
              }
           }
        }
     },
     "rancherCompose": {
        "version": "2",
        "services": {
           "launchbag-datascience2": {
              "scale": 1,
              "start_on_create": true
           }
        }
     },
     "startOnCreate": true,
     "binding": null
  }
```


- 참고 mmacpherson/datascience-notebook
  * https://github.com/mmacpherson/datascience-notebook/blob/master/Dockerfile
