---
date: 2022-12-15 18:16:42 +0900
title: Git 원격 저장소에 커밋된 파일 기록 지우기
categories: [ computer ]
tags: [ todo, git, image, metadata, exif, git-filter-repo, bfg, 100DaysToOffload ]
---
[처음으로 블로그에 쓴 글]({{< ref "/posts/2022-12-07-블로그" >}})에 실수를 했다.

생일 케이크 사진의 Exif 좌표가 저장된 채 GitHub 에 푸시해버렸다. 어처피 글 본 사람도 거의 없을테고 눈치챈 사람도 없을거 같아서 크게 걱정하진 않았는데 그래도 생각난 김에 삭제해보려고 한다.

## 시작하기 전
### 완전히 지울 순 없다!
비밀번호나 토큰 등 변경할 수 있는 정보를 올렸다면 아래 방법으로 기록을 지우지 말고 유출된 정보 자체를 변경해야한다. 지금부터 소개할 방법은 내가 소유 중인 저장소의 기록만 제거할 수 있다. 타인에 의해 포크 또는 클론 됐다면 모두 제거하는 건 현실적으로 불가능하다. *인터넷에 올려진 정보는 절대 삭제할 수 없다*.


### 누군가가 이미 내 저장소를 포크해갔다면?
이미 엎질러진 물을 전부 되담을 순 없다. 다만 포크된 원격 저장소가 GitHub 나 GitLab 등의 서비스를 통해 공개되어 있다면 [GitHub Support](https://support.github.com/request?tags=docs-generic) 등 수단을 통해 서비스 제공자에 문의해보자. 

~~정 안되면 DMCA...?~~


### 사용할 프로그램
Git 에는 브랜치를 다시 쓸 수 있는 [`git-filter-branch`](https://git-scm.com/docs/git-filter-branch) 가 있긴 하지만 속도나 호환성 문제가 있어 다른 도구를 추천[^git-filter-branch-warning]있다. 단순히 커밋만 지울 때는 [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) 나 [git-filter-repo](https://github.com/newren/git-filter-repo) 같은 특화된 프로그램을 사용하는게 좋다.

[^git-filter-branch-warning]: <https://git-scm.com/docs/git-filter-branch#_warning>

추가로, `git-filter-branch` 또는 `git-filter-repo` 를 사용하면 저장소 크기에 따라 작업 시간 매우 오래 걸릴 수 있다. 커밋 하나만 꼭 집어서 삭제하는 것이 아닌 기록 전체를 다시 작성하기 때문이다.


### 지역 저장소에서 주의할 점
수정된 파일이나 스태시가 존재하면 저장소가 꼬여 내용이 날아갈 수 있기 때문에 작업할 지역 저장소를 새로 클론하는 것이 좋다. 명령어 몇 줄이면 끝날 문제로 밤낮을 까고 싶지 않다면 깔끔하게 새로 클론한 뒤 작업하자...


### 원격 저장소에서 주의할 점
두 프로그램 모두 기록을 새로 만들어 기존 기록을 날려버리기 때문에 관련 커밋의 해시(SHA)와 종속된 커밋의 해시가 변경된다. 이 때 열려있던 풀 리퀘스트에 충돌 등의 문제가 생길 수 있으므로 [GitHub 공식 문서](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)에선 작업 전 모든 PR 을 병합하거나 닫는걸 추천하고 있다.


## 지역 저장소 클론하기
[`--mirror`](https://git-scm.com/docs/git-clone#Documentation/git-clone.txt---mirror) 인자를 통해 모든 참조를 그대로 가져온 새 지역 저장소를 만든다.

```
$ git clone --mirror git@github.com:toriato/toriato.github.io.git
$ cd toriato.github.io
```


## 파일 지우기
삭제하고픈 파일은 `rm` 이나 `git rm` 명령어로 지워준 뒤 지역 저장소에 커밋해야한다.

이 글의 목적은 **파일**을 지우는게 아닌 **커밋**을 지우는 것임을 알아두자

```
$ git rm /content/posts/2022-12-07-블로그/20221208_033408.jpg
$ git commit -m 'remove sensitive file'
```


## BFG 로 커밋 제거하기
<https://rtyley.github.io/bfg-repo-cleaner/>

BFG 는 다른 프로그램 대비 10배에서 720배 빠른 작업 속도를 자랑한다. 다만 모든 커밋에서 사용자가 지정한 파일 이름 또는 용량 조건이 일치할 때 제거하는 구조이기 때문에 **파일 지정시 경로는 사용할 수 없다**[^bfg-using-path]. 단순 삭제 작업이 아니라면 BFG 는 추천하지 않는다.

[^bfg-using-path]: <https://github.com/rtyley/bfg-repo-cleaner/issues/187#issuecomment-447995170>

먼저 실행하기 위해선 자바 런타임이 필요하다. 

공식 페이지에서 최신 `.jar` 파일을 받아 `java -jar bfg-<version>.jar` 명령어로 실행할 수 있는데 아래에선 편의를 위해 `bfg` 로 줄여쓰겠다.

```sh
$ bfg --delete-files 20221208_033408.jpg
```

커밋을 정상적으로 제거했다면 참조 기록(reflog)과 실제 파일의 데이터까지 지워준다.

```sh
$ git reflog expire --expire=now --all
$ git gc --prune=now --aggressive
```

원격 저장소에 푸시해주면 끝이다.

위에서 저장소를 클론할 때 `--mirror` 인자를 사용해 모든 참조를 가져와 수정한 뒤 푸시했기 때문에 시간이 조금 걸릴 수 있다.

```sh
$ git push --force
```

## git-filter-repo 로 커밋 제거하기
<https://github.com/newren/git-filter-repo>

*TODO: 언젠가 쓸 일이 있다면 사용 방법을 정리해 올려둘 예정*