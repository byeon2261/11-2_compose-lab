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
