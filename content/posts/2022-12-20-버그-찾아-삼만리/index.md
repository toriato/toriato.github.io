---
date: 2022-12-20 18:50:55 +0900
title: 버그 찾아 삼만리
categories: [ diary ]
tags: [ github, rust, rustdesk, bug, pipeline, xdg-desktop-portal, gstreamer, sway, 100DaysToOffload ]

ShowToc: true
---
GitHub 를 몇 년간 써오면서 다른 사람들과 협업하기 위해 사용해본적이 거의 없다.

나는 스스로 코딩을 못하는 축에 속한다고 생각한다. 내가 짠 코드들은 남들에게 보여주기 부끄럽고 오픈 소스 프로그램을 사용하다 버그를 발견하고 직접 고치거나 고칠 수 있는 아이디어가 떠올라도 내가 어딘가 실수하여 다른 사람들에게 지적받고 놀림 받을까봐 두려워 이슈나 풀 리퀘스트를 올리지 못했다. 그런 일이 생길 때마다 겨우 고졸 주제에, 취미 수준에 머물러 있는 지식으로 나댄다는 생각이 앞서 차마 그러지 못했다.

자기 비하는 여기까지 하고 아무튼...


## 버그
최근에 [RustDesk 관련된 글]({{< ref "/posts/2022-12-18-teamviewer-싫어-rustdesk-좋아" >}})을 쓰고 조금 써봤는데 Wayland 사용에 애로사항이 생겼었다. 버그를 발견하면 정확히 어떤 행동을 취했을 때 발생하는지 시험해보고 구글링으로 나와 비슷한 버그로 고생하고 있는 다른 사람이 있는지 찾아본다. 있으면 그 사람이 해결한 방법을 써보고 그래도 해결이 안된다면 그 때부터 내 손으로 찾아보기 시작한다.

일단 RustDesk 는 Wayland 를 지원하지 않았다. 작년 6월 5일에 올라온 [이슈](https://github.com/rustdesk/rustdesk/issues/56)를 시작으로 [H-M-H/Weylus](https://github.com/H-M-H/Weylus) 저장소 메인테이너의 도움[^weylus-maintainer-arrived]을 받아 지원을 시작한 것으로 보인다. 아직 완벽하진 않고 나이틀리 빌드에서 주요 기능만 지원하는 상태이다.

[^weylus-maintainer-arrived]: <https://github.com/rustdesk/rustdesk/issues/56#issuecomment-863654049>

[^setup-screen-sharing-in-sway]: <https://elis.nu/blog/2021/02/detailed-setup-of-screen-sharing-in-sway/>


## 헛걸음질
살면서 Rust 라는 언어를 손 대본적 없고 PipeWire, GStreamer 같은 라이브러리는 더욱 만져본적 없다. 애초에 네이티브나 임베디드, 시스템 같은 키워드와는 멀리 살아왔고 웹이나 고랭 같은 것만 깨작되던 나에게 이 버그는 누가 고쳐줄 때까지 손가락 빨며 기다려야할 버그였다. 아니... 버그였을터였다!

이유는 모르겠지만 이 버그를 무지성으로 고치고 싶어졌다. 일단 RustDesk 저장소에 [새 이슈](https://github.com/rustdesk/rustdesk/issues/2608)를 올리고 버그와 조금이라고 관련 있을 것으로 보이는 부분을 모두 확인했다.

### PipeWire?
PipeWire 는 영상과 음성을 처리하는 로우 레벨에서 동작하는 멀티미디어 프레임워크다. 낮은 지연 시간을 가지고 있고  PulseAudio, Jack, ALSA 와 GStreamer 기반의 어플리케이션을 지원한다.

`systemd` 에서 데몬으로 돌아가고 있는 `pipewire.service` 의 로그를 `journalctl` 로 확인해보면 아래와 같은 오류가 발생하는 것을 확인할 수 있다.

```log
Dec 20 19:22:24 laptop.remote systemd[742]: Started PipeWire Multimedia Service.
░░ Subject: A start job for unit UNIT has finished successfully
░░ Defined-By: systemd
░░ Support: https://lists.freedesktop.org/mailman/listinfo/systemd-devel
░░ 
░░ A start job for unit UNIT has finished successfully.
░░ 
░░ The job identifier is 213.
Dec 20 19:22:24 laptop.remote pipewire[93012]: [1:23:16.078432613] [93012]  INFO Camera camera_manager.cpp:299 libcamera v0.0.2
Dec 20 19:22:50 laptop.remote pipewire[93012]: pw.context: params Spa:Enum:ParamId:EnumFormat: 0:0 Invalid argument (input format (no more input formats))
Dec 20 19:22:50 laptop.remote pipewire[93012]: default: Object: size 96, type Spa:Pod:Object:Param:Format (262147), id Spa:Enum:ParamId:EnumFormat (3)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:mediaType (1), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 2        (Spa:Enum:MediaType:video)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:mediaSubtype (2), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 1        (Spa:Enum:MediaSubtype:raw)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:format (131073), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Choice: type Spa:Enum:Choice:None, flags 00000000 20 4
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Id 8        (Spa:Enum:VideoFormat:BGRx)
Dec 20 19:22:50 laptop.remote pipewire[93012]: pw.context: params Spa:Enum:ParamId:EnumFormat: 1:0 Invalid argument (output format (no more input formats))
Dec 20 19:22:50 laptop.remote pipewire[93012]: default: Object: size 296, type Spa:Pod:Object:Param:Format (262147), id Spa:Enum:ParamId:EnumFormat (3)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:mediaType (1), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 2        (Spa:Enum:MediaType:video)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:mediaSubtype (2), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 1        (Spa:Enum:MediaSubtype:raw)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:format (131073), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 8        (Spa:Enum:VideoFormat:BGRx)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:modifier (131074), flags 00000018
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Choice: type Spa:Enum:Choice:Enum, flags 00000000 96 8
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 72057594037927935
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 72057594037927935
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 144115206334822913
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 144115206334822657
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 144115206334806273
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 144115188080056833
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 144115188080056577
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 144115188075858433
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 144115188075858177
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Long 0
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:size (131075), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Rectangle 1920x1080
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:framerate (131076), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Fraction 0/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:maxFramerate (131077), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Choice: type Spa:Enum:Choice:Range, flags 00000000 40 8
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Fraction 60/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Fraction 1/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Fraction 60/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: default: Object: size 184, type Spa:Pod:Object:Param:Format (262147), id Spa:Enum:ParamId:EnumFormat (3)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:mediaType (1), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 2        (Spa:Enum:MediaType:video)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:mediaSubtype (2), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 1        (Spa:Enum:MediaSubtype:raw)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:format (131073), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Id 7        (Spa:Enum:VideoFormat:RGBx)
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:size (131075), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Rectangle 1920x1080
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:framerate (131076), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Fraction 0/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:   Prop: key Spa:Pod:Object:Param:Format:Video:maxFramerate (131077), flags 00000000
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:     Choice: type Spa:Enum:Choice:Range, flags 00000000 40 8
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Fraction 60/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Fraction 1/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: default:       Fraction 60/1
Dec 20 19:22:50 laptop.remote pipewire[93012]: pw.link: (67.0 -> 70.0) negotiating -> error (no more input formats)
```


### WirePlumber?

#### 디버그 로그 표시하기
`WIREPLUMBER_DEBUG` 값으로 표시할 로그 레벨을 설정할 수 있다[^set-wireplumber-log-level].
[^set-wireplumber-log-level]: <https://pipewire.pages.freedesktop.org/wireplumber/daemon-logging.html#debug-logging>

```
$ export WIREPLUMBER_DEBUG=D
$ systemctl --user import-environment WIREPLUMBER_DEBUG
$ systemctl --user restart wireplumber
```
```log
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiNode:0x55ac449a80a0> features changed 0x0 -> 0x1
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiNode:0x55ac449a80a0> activated item for node 71
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiNode:0x55ac449a80a0> handling item: rustdesk (71)
Dec 20 20:55:55 laptop.remote wireplumber[199890]: link rustdesk <-> xdg-desktop-portal-wlr passive:nil, passthrough:false, exclusive:nil
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiStandardLink:0x55ac448e34a0> create pw link: 68:72 (Spa:Enum:AudioChannel:UNK) -> 71:73 (Spa:Enum:AudioChannel:UNK)
Dec 20 20:55:55 laptop.remote wireplumber[199890]: 0x55ac44b22ed0: new 61 type PipeWire:Interface:Link/3 core-proxy:0x55ac448eb130, marshal:0x7f1fd123ccc0
Dec 20 20:55:55 laptop.remote wireplumber[199890]: 0x55ac448eb130: proxy id 61 bound 67
Dec 20 20:55:55 laptop.remote wireplumber[199890]: 0x55ac44b22ed0: id:61 bound:67
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpLink:67:0x55ac44b69700> features changed 0x0 -> 0x1
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a010> new WpGlobal:67 type:WpLink proxy:0x55ac44b69700
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a010> sync, seq 0x40000143, task <GTask:0x55ac44b2ad60>
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpLink:67:0x55ac44b69700> info, change_mask:0x7 [props,]
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpLink:67:0x55ac44b69700> features changed 0x1 -> 0x11
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a010> global:67 perm:0x1c8 type:PipeWire:Interface:Link/3 -> WpLink
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a010> reuse WpGlobal:67 type:WpLink proxy:0x55ac44b69700
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a110> global:67 perm:0x1c8 type:PipeWire:Interface:Link/3 -> WpLink
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a110> new WpGlobal:67 type:WpLink proxy:(nil)
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a110> sync, seq 0x40000071, task <GTask:0x55ac44b2aca0>
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiStandardLink:0x55ac448e34a0> features changed 0x0 -> 0x1
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiStandardLink:0x55ac448e34a0> activated si-standard-link
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiNode:0x55ac449a80a0> handling item: rustdesk (71)
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpSiNode:0x55ac449a80a0> ... already linked to proper target
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a110> done, seq 0x40000071, task <GTask:0x55ac44b2aca0>
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a110> exposing 1 new globals
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpLink:67:0x55ac44b69700> info, change_mask:0x1 []
Dec 20 20:55:55 laptop.remote wireplumber[199890]: 0x55ac448eb130: proxy 0x55ac44b22ed0 id:61: bound:67 seq:322 res:-22 (Invalid argument) msg:"no more input formats"
Dec 20 20:55:55 laptop.remote wireplumber[199890]: <WpCore:0x55ac4487a010> done, seq 0x40000143, task <GTask:0x55ac44b2ad60>
```


### `xdg-desktop-portal`?
sway 는 `xdg-desktop-portal`(이하 `xdp`)에서 사용하는 `XDG_CURRENT_DESKTOP` 값을 자동으로 설정하지 않는다[^xdg-is-not-set-by-sway][^xdg-is-not-set-by-sway-issue].

실행 전 수동으로 환경 변수를 잡아주던가 로그인 관리자를 통해 설정해야하는데 나는 전자를 사용했다.

[^xdg-is-not-set-by-sway]: <https://github.com/swaywm/sway/wiki#xdg_current_desktop-environment-variable-is-not-being-set>
[^xdg-is-not-set-by-sway-issue]: <https://github.com/swaywm/sway/pull/4876#issuecomment-570790128> 


#### `systemd` 환경 변수 적용하기
sway 를 통해 환경 변수를 적용하기 위해선 다음과 같은 설정을 추가하면 된다:
```
$ cat $XDG_CONFIG_HOME/sway/conf.d/90-fix-systemd-environment --style=plain 
exec systemctl --user import-environment XDG_SESSION_TYPE XDG_CURRENT_DESKTOP
exec dbus-update-activation-environment WAYLAND_DISPLAY
```

일시적으로 시험해보고 싶다면 다음 명령어를 사용하자:
```
$ export XDG_CURRENT_DESKTOP="sway" 
$ systemctl --user import-environment XDG_CURRENT_DESKTOP
```


#### 실행 중인 `pipewire` 와 `xdp` 재시작하기
`pipewire.service` 만 다시 시작하면 기존 `xdp` 프로세스는 사용할 수 없다. 

`systemdctl` 로 서비스를 같이 재시작하거나 아래 명령어로 실행 중인 프로세스를 죽이면 다시 인식된다.
``` 
$ killall xdg-desktop-portal xdg-desktop-portal-wlr
```


#### `xdp` 에서 사용 중인 `XDG_CURRENT_DESKTOP` 값 가져오기[^source-of-getting-xcd-of-xdp]
[^source-of-getting-xcd-of-xdp]: <https://github.com/emersion/xdg-desktop-portal-wlr/wiki/%22It-doesn't-work%22-Troubleshooting-Checklist>

정상적으로 `sway` 로 잡혀있어도 동일한 오류가 발생하므로 `xdp` 는 범인이 아니였다...

무엇보다 모질라 재단의 [`getUserMedia` 테스트 페이지](https://mozilla.github.io/webrtc-landing/gum_test.html)가 정상적으로 동작하니 애초에 `xdp` 와는 관계 없었던 것이다.

```
$ systemctl --user show-environment | grep XDG
...
XDG_CURRENT_DESKTOP=sway
XDG_SESSION_TYPE=wayland
...
```
```
$ < "/proc/$(pidof xdg-desktop-portal)/environ" tr '\0' '\n' | grep '^XDG_CURRENT_DESKTOP='
XDG_CURRENT_DESKTOP=sway
```


## 범인은 따로 있었다!
계속 검색을 해보니 이 문제가 Wayland 환경 중에서도 sway 에서만 발생한다는 사실을 알게됐다[^weylus-pixel-format-issue]. sway 는 `BGRx` 를 지원하지 않는데[^sway-does-not-support-bgrx] RustDesk 는 `BGRx` 만 지원해 `pipewire` 에서 잘못된 입력 포맷 오류가 발생했던거다!

[^weylus-pixel-format-issue]: <https://github.com/H-M-H/Weylus/issues/155#issuecomment-1120327869>
[^sway-does-not-support-bgrx]: *TODO: 아무리 찾아봐도 출처가 어디인지 모르겠다*

```rs
let appsink = sink
    .dynamic_cast::<AppSink>()
    .map_err(|_| GStreamerError("Sink element is expected to be an appsink!".into()))?;
appsink.set_caps(Some(&gst::Caps::new_simple(
    "video/x-raw",
    &[("format", &"BGRx")],
)));
```
RustDesk 소스 중 `pipewire` 와 관련된 코드를 담고 있는 [libs/scrap/src/wayland/pipewire.rs](https://github.com/rustdesk/rustdesk/blob/d910e7ad96c8389e1bc3f2a2f0f3c13d2b43485f/libs/scrap/src/wayland/pipewire.rs#L147-L153) 파일을 열어보면 [AppSink](https://gstreamer.freedesktop.org/documentation/app/appsink.html?gi-language=c) 에 등록된 [Caps](https://gstreamer.freedesktop.org/documentation/additional/design/caps.html?gi-language=c) 가 `BGRx` 하나 뿐인 걸 알 수 있다. RustDesk 가 GStreamer 의 [Caps negotition](https://gstreamer.freedesktop.org/documentation/plugin-development/advanced/negotiation.html?gi-language=c) 과정에서 sway 에는 없는 `BGRx` 포맷을 요청하니 `no more input formats` 오류를 반환한 것이다.

이 문제를 해결하려면 RustDesk 에 `RGBx` 를 처리할 수 있는 코드를 추가해주면 된다. 나는 [H-M-H/Weylus](https://github.com/H-M-H/Weylus) 저장소의 이슈[^weylus-pixel-format-issue]를 보고 관련 커밋[^weylus-rgbx-issue-commit-initial][^weylus-rgbx-issue-commit-fix-color]을 거의 복붙하다시피 참고해 문제를 해결했다.

[^weylus-rgbx-issue-commit-initial]: <https://github.com/H-M-H/Weylus/commit/55301e05416171afb359fe9580b7038f22038f9d>
[^weylus-rgbx-issue-commit-fix-color]: <https://github.com/H-M-H/Weylus/commit/d399fb4864c25438639e4c01b1be5d823df1aaa3>

```rs
let appsink = sink
    .dynamic_cast::<AppSink>()
    .map_err(|_| GStreamerError("Sink element is expected to be an appsink!".into()))?;
let mut caps = gst::Caps::new_empty();
caps.merge_structure(gst::structure::Structure::new(
    "video/x-raw",
    &[("format", &"BGRx")],
));
caps.merge_structure(gst::structure::Structure::new(
    "video/x-raw",
    &[("format", &"RGBx")],
));
appsink.set_caps(Some(&caps));
```

Caps 에 지원하는 픽셀 포맷을 넣는거야 단순히 복붙으로 해결할 수 있지만 이대로 실행하면 `BGRx` 값이 그대로 사용되기 때문에 색이 뒤집혀진다. 참고한 저장소에선 C 언어로 변환 과정을 구현했는데 운 좋게도 RustDesk 소스 중 `libyuv` 를 통해 [`RGBx` 포맷을 `i420` 으로 변환하는 함수](https://github.com/rustdesk/rustdesk/blob/364387028734c1dc603d6ff2a7924c9cd54247e3/libs/scrap/src/common/convert.rs#L192-L213)가 있었다.


## PR 올리기
<https://github.com/rustdesk/rustdesk/pull/2612>

대충 코드를 수정하고 내친김게 Gstreamer 버전도 업데이트한 뒤 새 버전에서 바뀐 메소드를 고쳤다. 정상적으로 동작하는 것을 확인했지만 영어에 자신이 없어 최근에 [OpenAI 에서 공개한 ChatGPT](https://openai.com/blog/chatgpt/) 를 사용해 번역했다.

{{< figure src="chatgpt-doing-a-thing.png" >}}
