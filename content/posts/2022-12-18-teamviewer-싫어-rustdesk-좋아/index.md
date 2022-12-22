---
date: 2022-12-18 18:48:30 +0900
title: TeamViewer 싫어, RustDesk 좋아!
categories: [ computer ]
tags: [ todo, remote-desktop, rdp, teamviewer, rustdesk, docker, 100DaysToOffload ]

ShowToc: true
---

외부에서 우리집 윈도우에 접속하기 위해 RDP 를 사용하고 있었는데 꽤 많은 문제가 있었다.

가정에서 사용하는 윈도우 라이선스로 실행할 수 있는 RDP 는 기능에 제한이 많아 [RDPWrap](https://github.com/stascorp/rdpwrap) 을 사용하고 있었는데 RDPWrap 은 `termsrv.dll` 파일의 인자를 오프셋으로 수정해 불러오는 형태로 구현하다보니 윈도우 업데이트 등의 이유로 `termsrv.dll` 파일이 변경되면 `rdpwrap.ini` 파일에 수정된 오프셋을 직접 찾아 넣어줘야하는 불편함이 있었다. 그렇다고 윈도우 업데이트를 꺼버리기엔 보안 이슈나 버그가 발생할까 무섭고 수동으로 실행하기엔 너무 귀찮아 RDP 를 완전히 대체할 프로그램을 찾을 필요가 있었다.

[AutoDesk](https://anydesk.com) 나 [TeamViewer](https://www.teamviewer.com) 같은 유료 또는 로그인이 필요한 프로그램은 당연히 제외했고 [Chrome Desktop Remote](https://remotedesktop.google.com/) 도 내 취향에 맞지 않아 제외했다. 그러던 도중 RustDesk 를 발견했다.

## RustDesk
<https://rustdesk.com/>  
<https://github.com/rustdesk/rustdesk>

RustDesk 는 TeamViewer 와 비슷한 서버-클라이언트 구조를 가진 원격 프로그램을 Rust 로 구현한 오픈 소스 프로젝트다.


### 동작 구조
{{< figure 
  src="how-rustdesk-works.png"
  class="center"
  caption="[이미지 출처](https://github.com/rustdesk/rustdesk/issues/594#issuecomment-1148133777)"
>}}


## 서버 설치 및 설정
서버를 호스팅하기 위해선 ID/랑데부 서버(`hbbs`)와 릴레이 서버(`hbbr`)를 실행해야한다.

다행히 도커 이미지가 있으니 홈 서버에서 사용하던 도커 컴포즈에 해당 컨테이너만 설정해주면 쉽게 실행할 수 있다.


### 포트 열기
따로 설정하지 않았다면 `hbbs`는 21115 (TCP), 21116 (TCP/UDP) 그리고 21118(TCP) 포트를 사용하고 `hbbr` 은 21117 (TCP) 와 21119 (TCP) 를 사용한다.

포트와 관련된 코드는 [`rustdesk/rustdesk-server`](https://github.com/rustdesk/rustdesk-server) 저장소의 [`/src/utils.rs` 파일](https://github.com/rustdesk/rustdesk-server/blob/85af668a4f746a8138cbc0d26e2eba66ab95cfa0/src/utils.rs#L113-L119)에서 확인할 수 있다.

- `21115/TCP` NAT 종류 테스트
- `21116/TCP` [TCP hole punching](https://en.wikipedia.org/wiki/TCP_hole_punching) 및 서비스 연결
- `21117/TCP` 릴레이 서비스
- `21116/UDP` ID 등록 및 Heartbeat
- `21118-21119/TCP` 웹 클라이언트 연결 (사용하지 않는다면 닫아도 괜찮음)

#### API
공식 문서에 적혀있지 않는 포트 사용이 확인됐다. 나중을 위한 포트[^21114-tcp-port-for-future-use]라는 답변을 이슈에서 찾았는데 실제로는 현 버전에서 API 와 관련된 웹 포트로 사용하고 있었다.

[^21114-tcp-port-for-future-use]: <https://github.com/rustdesk/rustdesk/issues/518#issuecomment-1126774740>

해당 포트는 `PORT_FOR_API` 환경 변수로 수정할 수 있다. 클라이언트는... 잘 모르겠다.


#### 공유기 포트포워딩
{{< figure src="port-forwarding-from-router.png" >}}


#### `firehol` 방화벽 포트 추가
```sh
#!/usr/bin/env bash

...

server_rustdesk_ports="tcp/21115,21116,21117,21118,21119 udp/21116"
client_rustdesk_ports="default"

interface4 enp2s0 lan
  policy reject

  server all accept src "${home_ips}"
  server "ssh http https rustdesk" accept

  client all accept
```


### 도커 컴포즈 설정
```yaml
  # 릴레이 서버
  rustdesk-hbbr:
    image: docker.io/rustdesk/rustdesk-server:latest
    command: hbbr
    restart: unless-stopped
    ports:
      - 21117:21117
      - 21119:21119
    volumes: [ ./apps/rustdesk/hbbr:/root ]

  # ID/랑데부 서버
  rustdesk-hbbs:
    image: docker.io/rustdesk/rustdesk-server:latest
    command: "hbbs -r rustdesk-hbbr:21117"
    restart: unless-stopped
    depends_on: [ rustdesk-hbbr ]
    ports:
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21118:21118
    volumes: [ ./apps/rustdesk/hbbs:/root ]
```


#### 로그
서비스가 정상적으로 실행됐다면 아래와 같은 로그가 출력된다.


##### `hbbr`
```sh
❯ dcl --no-log-prefix rustdesk-hbbr
[2022-12-18 10:34:49.283365 +00:00] INFO [src/relay_server.rs:60] #blacklist(blacklist.txt): 0
[2022-12-18 10:34:49.283374 +00:00] INFO [src/relay_server.rs:75] #blocklist(blocklist.txt): 0
[2022-12-18 10:34:49.283376 +00:00] INFO [src/relay_server.rs:81] Listening on tcp 0.0.0.0:21117
[2022-12-18 10:34:49.283379 +00:00] INFO [src/relay_server.rs:83] Listening on websocket 0.0.0.0:21119
[2022-12-18 10:34:49.283380 +00:00] INFO [src/relay_server.rs:85] Start
[2022-12-18 10:34:49.283394 +00:00] INFO [src/relay_server.rs:104] DOWNGRADE_THRESHOLD: 0.66
[2022-12-18 10:34:49.283398 +00:00] INFO [src/relay_server.rs:113] DOWNGRADE_START_CHECK: 1800s
[2022-12-18 10:34:49.283399 +00:00] INFO [src/relay_server.rs:122] LIMIT_SPEED: 4Mb/s
[2022-12-18 10:34:49.283401 +00:00] INFO [src/relay_server.rs:132] TOTAL_BANDWIDTH: 1024Mb/s
[2022-12-18 10:34:49.283403 +00:00] INFO [src/relay_server.rs:146] SINGLE_BANDWIDTH: 16Mb/s
```


##### `hbbs`
```sh
❯ dcl --no-log-prefix rustdesk-hbbs
[2022-12-18 10:34:49.407746 +00:00] INFO [src/common.rs:132] Private/public key written to id_ed25519/id_ed25519.pub
[2022-12-18 10:34:49.407760 +00:00] INFO [src/peer.rs:82] DB_URL=./db_v2.sqlite3
[2022-12-18 10:34:49.418009 +00:00] INFO [src/rendezvous_server.rs:98] serial=0
[2022-12-18 10:34:49.418023 +00:00] INFO [src/common.rs:40] rendezvous-servers=[]
[2022-12-18 10:34:49.418026 +00:00] INFO [src/rendezvous_server.rs:100] Listening on tcp/udp 0.0.0.0:21116
[2022-12-18 10:34:49.418028 +00:00] INFO [src/rendezvous_server.rs:101] Listening on tcp 0.0.0.0:21115, extra port for NAT test
[2022-12-18 10:34:49.418029 +00:00] INFO [src/rendezvous_server.rs:102] Listening on websocket 0.0.0.0:21118
[2022-12-18 10:34:49.418043 +00:00] INFO [libs/hbb_common/src/udp.rs:35] Receive buf size of udp 0.0.0.0:21116: Ok(212992)
[2022-12-18 10:34:49.418067 +00:00] INFO [src/rendezvous_server.rs:137] mask: None
[2022-12-18 10:34:49.418072 +00:00] INFO [src/rendezvous_server.rs:138] local-ip: ""
[2022-12-18 10:34:49.418274 +00:00] INFO [src/common.rs:40] relay-servers=["rustdesk-hbbr:21117"]
[2022-12-18 10:34:49.418305 +00:00] INFO [src/rendezvous_server.rs:154] ALWAYS_USE_RELAY=N
[2022-12-18 10:34:49.418314 +00:00] INFO [src/rendezvous_server.rs:174] Start
```


#### 폴더 구조
서비스가 정상적으로 실행됐다면 아래와 같은 폴더 구조와 파일이 생성된다.


```
aeon in 🌐 miracle in miracle/apps/rustdesk on  master [✘+?]
❯ ls -R
drwxr-xr-x - aeon 12-18 19:34 hbbr
drwxr-xr-x - aeon 12-18 19:34 hbbs

./hbbr:

./hbbs:
.rw-r--r-- 4.1k aeon 12-18 19:34 db_v2.sqlite3
.rw-r--r--  33k aeon 12-18 19:34 db_v2.sqlite3-shm
.rw-r--r--  41k aeon 12-18 19:34 db_v2.sqlite3-wal
.rw-r--r--   88 aeon 12-18 19:34 id_ed25519
.rw-r--r--   44 aeon 12-18 19:34 id_ed25519.pub
```


### 공개 키
다른 사용자(단말기)가 내 서버를 사용하기 위해선 `hbbs` 의 작업 디렉터리 속에 자동으로 생성된 `id_ed25519.pub` 파일 속 인코딩된 공개 키 문자열이 필요하다.


#### 공개 키 사용 안하기
`hbbs` 와 `hbbr` 를  `-k _` 인자와 함께 실행하면 공개 키 인증을 사용하지 않는다.


## 클라이언트 설치 및 설정
윈도우와 맥은 [GitHub 저장소 릴리즈 탭](https://github.com/rustdesk/rustdesk/releases)에서, 리눅스는 패키지 매니저나 통하거나 [소스로부터 빌드해](https://github.com/rustdesk/rustdesk#build) 설치할 수 있고 안드로이드는 [F-Droid](https://f-droid.org/packages/com.carriez.flutter_hbb/) 등[^rustdesk-android-client-from-github-release][^rustdesk-android-client-from-google-play]에서, iOS 는 [앱 스토어](https://apps.apple.com/app/rustdesk-remote-desktop/id1581225015)를 통해 설치할 수 있다.

[^rustdesk-android-client-from-github-release]: GitHub 릴리즈 탭에서도 APK 파일을 받을 수 있다: <https://github.com/rustdesk/rustdesk/releases>
[^rustdesk-android-client-from-google-play]: 추천하진 않지만 구글 플레이서도 받을 순 있다: <https://play.google.com/store/apps/details?id=com.carriez.flutter_hbb>


### Windows
윈도우 환경에서 사용할 수 있는 CLI 패키지 매니저인 [`scoop`](https://github.com/ScoopInstaller/Scoop) 의 포크, [`shovel`](https://github.com/Ash258/Scoop-Core) 를 사용해 설치했다.

```
PowerShell 7.3.0

~
❯ shovel search rustdesk
Searching in local buckets...
extras/rustdesk
  Version: 1.1.9
  Description: An open-source remote desktop software, written in Rust.
  Shortcuts:
    - RustDesk.exe > RustDesk

~ took 1m42s
~
❯ shovel install rustdesk
Installing 'rustdesk' (1.1.9) [64bit] [extras]
WARN  By installing you accept following license: AGPL-3.0-only (https://spdx.org/licenses/AGPL-3.0-only.html)
Starting download with aria2 ...
Download: Download Results:
Download: gid   |stat|avg speed  |path/URI
Download: ======+====+===========+=======================================================
Download: 8f267c|OK  |    29MiB/s|C:/Users/totor/scoop/cache/rustdesk#1.1.9#https_github.com_rustdesk_rustdesk_releases_download_1.1.9_rustdesk-1.1.9-windows_x64.zip
Download: Status Legend:
Download: (OK):download completed.
Checking hash of rustdesk-1.1.9-windows_x64.zip ... ok.
Extracting rustdesk-1.1.9-windows_x64.zip ... done.
Running pre-install script...
Linking ~\scoop\apps\rustdesk\current => ~\scoop\apps\rustdesk\1.1.9
Creating shortcut for RustDesk (RustDesk.exe)
'rustdesk' (1.1.9) was installed successfully!

~ took 3s
```


### Linux
작업용 노트북에 설치된 아치 리눅스에선 AUR[^aur-rustdesk][^aur-rustdesk-bin] 를 통해 쉽게 설치할 수 있었다. 다만 내 노트북에선 Wayland 컴포지터인 [`sway`](https://github.com/swaywm/sway) 를 사용하고 있는데 RustDesk 는 Wayland 를 완벽히 지원하지 않는다.

[^aur-rustdesk]: <https://aur.archlinux.org/packages/rustdesk>
[^aur-rustdesk-bin]: <https://aur.archlinux.org/packages/rustdesk-bin>


#### Wayland
다행히 메인테이너가 이 문제를 인지하고 있고 현재 작업 중[^wayland-support-issue]이다. 완벽하진 않으나 나이틀리 빌드[^nightly-release][^aur-rustdesk-nightly]에선 어느정도 지원하는 것을 확인할 수 있었다. 다른 원격 장치에 접속했을 때 키보드 입력이 안되는 문제는 해결됐으나 원격 데스크톱은 여전히 열리지 않는다.

[^nightly-release]: <https://github.com/rustdesk/rustdesk/releases/tag/nightly>
[^aur-rustdesk-nightly]: <https://aur.archlinux.org/packages/rustdesk-nightly>
[^wayland-support-issue]: <https://github.com/rustdesk/rustdesk/issues/56>


## 사용 후기
- 아직 Wayland 를 지원하지 않음
- 일반 사용자의 오류 보고가 쉽지 않음, 특히 클라이언트는 사실상 불가능함[^logging-with-env-variable]
- 공식 문서의 많은 부분이 현 버전과 차이가 있고 뭔가 많이 빠져있음
- 구현 안된 기능이 많고 버그도 산더미인데 벌써부터 유료 버전을 생각하고 있음[^pro-version]
- RDP 나 Chrome Remote Deskop 등 다른 원격 데스크톱 소프트웨어 대비 압축 관계 없음 화질과 속도 모두 나쁨

[^logging-with-env-variable]: <https://github.com/rustdesk/rustdesk/issues/515#issuecomment-1126688734>
[^pro-version]: <https://github.com/rustdesk/rustdesk/issues/509#issuecomment-1125744953>

*TODO: 지속적으로 작성할 예정*