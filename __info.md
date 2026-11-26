# 서비스 배포 및 자동화 운영

## Docker Compose 기본 실습

<aside>
💡

**Compose는 컨테이너의 실행 조건을 파일로 기록하고 함께 관리한다.**
같은 구성 파일을 웹 한 개에서 두 서비스로 확장하며 실행·통신·파일 저장을 확인한다.
호스트 파일을 연결하는 바인드 마운트와, 컨테이너가 바뀌어도 데이터를 남기는 이름 있는 볼륨을 구별한다.

</aside>

### Compose의 실행 선언

앞서 일정 조정 서비스의 웹·API를 실행하며 이미지·포트·네트워크를 명령에 지정했다. **Docker Compose**는 이 실행 조건을 `compose.yaml`에 기록하고 여러 컨테이너를 함께 관리한다.

### 단일 서비스의 실행과 상태

#### 현재 구성

컨테이너를 만들고 백그라운드에서 실행한 뒤 상태를 확인한다. `-d`는 실행을 시작한 뒤 터미널로 돌아오는 옵션이다.

```powershell
docker compose up -d
docker compose ps
```

필요한 이미지가 없으면 가져온 뒤 네트워크와 웹 컨테이너를 만든다. `ps`에서 `web`의 `STATUS`가 `Up`인지, `PORTS`가 `127.0.0.1:8082->80/tcp`인지 확인한다. Compose ps

브라우저에서 `http://localhost:8082/`의 Nginx 기본 화면을 연 뒤 요청 로그를 읽는다. `--tail 20`은 지정한 서비스 `web`의 마지막 20줄을 보여 준다.

```powershell
docker compose logs --tail 20 web
```

로그 앞의 `web-1` 같은 표시는 어느 컨테이너의 기록인지 알려 준다. 요청 경로 `GET /`와 정상 응답 `200`을 화면 요청에 연결한다. Compose 명령에서는 YAML에 적은 서비스 이름 `web`으로 대상을 지정한다. Compose logs · Compose 시작 안내

### 바인드 마운트와 호스트 파일

호스트에서 편집하는 HTML을 웹 컨테이너에 연결한다. 이렇게 **호스트의 파일이나 폴더를 컨테이너 경로에 연결하는 설정**이 바인드 마운트다.

#### 현재 구성과 화면 파일

```yaml
...
    volumes:
      - ./site:/usr/share/nginx/html:ro
```
./site(호스트): /usr/share/nginx/html:ro(컨테이너) = 둘을 서로 연결시킨다.

#### 연결 적용과 변경 반영

마운트 설정을 적용하고, 출력된 웹 컨테이너 ID를 기록한다.

```powershell
docker compose config --quiet
docker compose up -d
docker compose ps -q web
```

브라우저에서 `http://localhost:8082/`를 새로 고쳐 직접 만든 화면을 확인한다. 이어서 호스트 `site/index.html`의 `<h1>` 문구를 `바인드 마운트 변경 확인`으로 편집·저장한 뒤 브라우저에서 강력 새로 고침한다.

화면 문구가 바뀌면 `docker compose ps -q web`을 다시 실행해 앞의 ID와 비교한다. **ID는 그대로인데 화면은 바뀐다.** 같은 컨테이너가 연결된 호스트 폴더의 새 내용을 읽은 것이다. 바인드 마운트

compose.yaml에 내용을 추가해주고 nginx.conf파일을 추가한다.

```yaml
name: compose-lab

services:
  ...

  files:
    image: python:3.12-slim
    command:
      - sh
      - -c
      - mkdir -p /data && exec python -m http.server 8000 --directory /data
```

compose up을 실행하면 터미널 files이 추가된다.

#### 메모 파일의 전체 전달 경로

이번에는 돌아온 내용을 분명하게 확인할 수 있도록 메모를 만든다. `files` 안의 `/data/memo.txt`에 내용을 쓰고 읽는다. `python -c`는 뒤의 문자열을 Python 코드로 실행한다.

```powershell
docker compose exec files python -c "from pathlib import Path; Path('/data/memo.txt').write_text('container layer memo', encoding='utf-8')"
docker compose exec files python -c "from pathlib import Path; print(Path('/data/memo.txt').read_text(encoding='utf-8'))"
```

첫 명령이 파일을 쓰고, 두 번째 명령이 `container layer memo`를 출력한다.

이제 브라우저에서 `http://localhost:8082/files/memo.txt`를 열거나 화면의 파일 링크를 누른다. 이번 요청은 **브라우저 → 호스트 8082번 → 웹 컨테이너의 Nginx(80번) → 파일 서버(8000번) → 메모 파일**을 거친다. 브라우저에 같은 메모 내용이 표시되는지 확인한다.

지금 생성한 파일은 compose가 내려가면 컨테이너가 삭제되고 파일도 같이 삭제된다.

### 이름 있는 볼륨과 데이터 수명

#### 이름 있는 볼륨의 연결

다음에는 파일을 컨테이너와 별도 수명의 저장 공간에 둔다. 먼저 현재 프로젝트를 정리한다.

```powershell
docker compose down
```

컨테이너와 기본 네트워크가 제거된 상태에서 `compose.yaml`을 아래 전체 내용으로 저장한다. 추가할 곳은 `files.volumes`와 최상위 `volumes`다.

```yaml
name: compose-lab

services:
  ...
    volumes:
      - lab-data:/data

volumes:
  lab-data:
```

`lab-data:/data`는 Docker가 관리하는 **이름 있는 볼륨**을 파일 서버의 `/data`에 연결한다. 실제 이름은 프로젝트 이름이 붙은 `compose-lab_lab-data`이며, 호스트의 `site` 폴더와는 별도 저장 공간이다.

compose up을 하면 Volumes에 compose-lab_lab-data이 생성된다. 다시 파일을 생성하고 compose삭제 후 재실행해되 파일을 확인 할 수 있다.

#### 볼륨 삭제와 저장 범위

현재 화면 제목 `바인드 마운트 변경 확인`과 메모 내용 `named volume memo`를 기록한다. 호스트의 `site/index.html`과 볼륨의 `/data/memo.txt`가 삭제 명령의 영향을 어떻게 받는지 대조한다.

현재 프로젝트 이름이 `compose-lab`인지 확인하고 볼륨까지 제거한다. `down -v`는 이 프로젝트의 컨테이너·네트워크와 선언된 볼륨을 제거하며, 호스트의 바인드 원본과 사용한 이미지는 남긴다. Compose down의 제거 범위

`volume ls`의 `--filter label=com.docker.compose.project=compose-lab`는 Compose의 프로젝트 표식으로 이 랩의 볼륨만 골라 보여 준다.

```powershell
docker compose down -v
docker volume ls --filter label=com.docker.compose.project=compose-lab
```

목록에 열 제목만 있고 데이터 행이 없으면 랩 볼륨이 제거된 상태다.

같은 구성으로 다시 실행해 메모 응답과 실제 데이터 폴더를 확인한다.

```powershell
docker compose up -d
curl.exe -i http://localhost:8082/files/memo.txt
docker compose exec files python -c "from pathlib import Path; print(list(Path('/data').iterdir()))"
docker volume inspect compose-lab_lab-data
```

메모 요청은 404이고 `/data` 목록은 `[]`이며, 같은 이름으로 생성된 새 볼륨은 비어 있다. 브라우저의 `http://localhost:8082/`를 강력 새로 고침하면 호스트에 남아 있는 화면 제목 `바인드 마운트 변경 확인`은 그대로 보인다.

| 동작 | 컨테이너 | 이 랩의 이름 있는 볼륨 | 호스트의 바인드 원본 |
| --- | --- | --- | --- |
| `stop` → `start` | 같은 컨테이너 재사용 | 유지 | 유지 |
| `down` → `up` | 제거 후 새로 생성 | 유지 후 재연결 | 유지 후 재연결 |
| `down -v` | 제거 | 제거 | 유지 |

### 실습 · 새 컨테이너와 공유 볼륨

웹 서버는 요청을 기다리며 계속 실행되지만, 파일 생성처럼 한 번 수행하고 끝나는 작업도 컨테이너로 실행할 수 있다. 이런 작업은 끝난 뒤 `Exited (0)`으로 표시되면 정상 종료한 것이다.

현재 `web`·`files`는 실행 중이고, 파일 서버의 `/data`에 연결된 `lab-data`는 비어 있다. 호스트의 화면 파일은 그대로 남아 있다. 여기에 메모를 한 번 만드는 `writer` 서비스를 추가한다.

해당 writer컨테이너는 파일생성 명령어 실행 후 종료된다.