---
type: project
title: GyuOS 개발 기록
source: Notion
source_url: https://app.notion.com/p/3f06bc51992b815b96adf6a1fc92767b?pvs=204
notion_last_edited: 2026-10-05T23:12:39.169Z
verification: unverified
imported: 2026-11-05
tags:
  - 프로젝트
  - 운영체제
  - Rust
---

# GyuOS 개발 기록

> Notion 원본: https://app.notion.com/p/3f06bc51992b815b96adf6a1fc92767b?pvs=204
> Notion 최종 수정: 2026-10-05T23:12:39.169Z
> 검증 상태: unverified

# 프로젝트 개요

**GyuOS**는 Linux 배포판을 커스터마이징하는 프로젝트가 아니라, **Codex를 활용해 x86\_64 UEFI 환경에서 밑바닥부터 Windows풍 데스크톱 OS를 어디까지 구현할 수 있는지 실험하는 개인 Hobby OS 프로젝트**다.

- 개발 호스트: Windows
- 주요 언어: Rust
- 실행/검증 환경: VMware Workstation
- 기반 OS: 없음 — Linux kernel, KDE, GNOME 등을 런타임 기반으로 사용하지 않음
- 부팅: x86\_64 UEFI
- 그래픽: UEFI framebuffer 직접 렌더링
- 현재 이미지: UEFI raw IMG + VMware VMDK

> 
초기에는 Debian + live-build + KDE Plasma + Calamares 기반 배포판으로 설계했으나, 프로젝트 의도가 “Linux 꾸미기”가 아니라 “자체 OS 구현”임을 명확히 한 뒤 해당 구조를 전부 제거하고 자체 kernel 방식으로 전환했다.

# 개발 원칙

- Windows의 일반적인 데스크톱 UX는 참고하되 Microsoft의 소스 코드, 바이너리, 로고, 아이콘, 폰트, 배경화면 등 자산은 사용하지 않는다.
- 실제 구현되지 않은 기능이나 시스템 정보를 가짜 값으로 표시하지 않는다.
- VMware 우선으로 개발하며 실제 하드웨어 지원은 현재 목표가 아니다.
- GUI를 빠르게 구현하고, 필요한 kernel 기능을 실제 요구가 생길 때 아래 계층에 추가하는 방식으로 진행한다.

# 진행 기록

## 1. 자체 UEFI Kernel 전환

기존 Debian/live-build/KDE/Calamares 관련 구현을 제거했다.

최초 자체 OS 버전에서 다음을 확인했다.

- UEFI 디스크 인식
- 자체 GyuOS Kernel 진입
- framebuffer 초기화
- GyuOS 로고와 배경 렌더링
- Test Window
- Taskbar / Start 버튼
- mouse pointer 렌더링
- VMware 실제 UEFI 부팅 성공

빌드 명령:

```powershell
.\build.ps1
.\build.ps1 -VmwareImage
.\run.ps1
```

산출물은 `gyuos-uefi.img`, `gyuos-uefi.vmdk` 형태로 생성한다.

## 2. Renderer / Input 최적화

초기 구현에서는 PS/2 mouse packet마다 1024×768 전체 Desktop을 실제 framebuffer에 직접 다시 그려 화면 flickering과 mouse latency가 발생했다.

개선 내용:

- 16 MiB scene buffer + composite buffer
- 완성된 frame만 실제 framebuffer로 전달
- pitch/stride 및 RGB/BGR/U8 pixel format 처리
- frame당 allocation 제거
- 고정 크기 damage tracker
- cursor 이동 시 이전/새 cursor 영역만 damage 처리
- Start Menu 및 key 표시 영역 부분 갱신
- PS/2 상태를 `Empty / Pending / Event`로 분리
- render 전에 준비된 PS/2 input을 모두 처리
- PS/2 mouse sample rate 200 Hz
- sleep/debounce/frame cap/좌표 scaling 제거

현재 PS/2는 polling 방식이다. IDT/PIC/APIC 및 IRQ event queue가 아직 없기 때문에 IRQ1/IRQ12 방식은 이후 단계로 남겨두었다.

## 3. Window Manager / Desktop Interaction

자체 Window Manager를 구현했다.

- 최대 4개 Window 동시 관리
- title bar drag
- Window focus
- z-order
- 활성/비활성 Window 시각 구분
- Close
- Minimize
- Maximize / Restore
- Taskbar 실행 Window 표시
- Taskbar 클릭으로 minimize/restore/focus
- Start Menu에서 Files / Terminal / Settings 실행
- Start Menu 바깥 클릭 시 닫기
- 최소화 앱을 Start Menu에서 실행하면 restore/focus
- Test/Terminal Window keyboard input 표시

Window 이동 시 전체 Desktop을 다시 그리지 않고 이전/새 Window bounds만 damage 처리하며 해당 영역을 z-order 순서로 다시 composite한다.

## 4. GyuOS Terminal

Window Manager 위에서 실제 Terminal 애플리케이션을 구현했다.

Terminal 구조:

- 독립적인 `TerminalState`
- 입력 buffer
- 8개 command history
- 64줄 output ring buffer
- 자동 vertical scrolling
- Terminal별 독립 상태 slot
- UI와 분리된 shell parser

PS/2 Set-1 keyboard decoder:

- 영문
- 숫자
- 기본 기호
- Shift / Caps Lock
- Backspace
- Enter
- Up / Down history navigation

현재 shell command:

```txt
help
ver
clear
echo <text>
mem
uptime
windows
sysinfo
reboot
shutdown
```

`reboot`는 PS/2 controller reset을 이용해 구현했다.

아직 실제 timer, heap, ACPI shutdown 기반이 없으므로 해당 정보는 임의 값을 만들지 않고 `not available`로 표시한다.

Terminal 입력 시 전체 화면이 아니라 해당 Window content 영역만 damage 처리한다.

# 현재 아키텍처

```txt
VMware UEFI
    ↓
GyuOS Boot Image
    ↓
GyuOS Rust Kernel
    ↓
UEFI Framebuffer
    ↓
Scene Buffer / Composite Buffer
    ↓
Damage Tracker
    ↓
Desktop Shell
    ├─ Window Manager
    ├─ Taskbar
    ├─ Start Menu
    └─ Mouse / Keyboard Input
         ↓
Applications
    ├─ Test Window
    ├─ Files
    ├─ Terminal
    └─ Settings
```

# 현재 구현 상태

<table fit-page-width="true" header-row="true">
<tr>
<td>
영역
</td>
<td>
상태
</td>
<td>
비고
</td>
</tr>
<tr>
<td>
UEFI Boot
</td>
<td>
완료
</td>
<td>
VMware cold boot 확인
</td>
</tr>
<tr>
<td>
자체 Rust Kernel
</td>
<td>
완료
</td>
<td>
Linux kernel 미사용
</td>
</tr>
<tr>
<td>
Framebuffer Renderer
</td>
<td>
완료
</td>
<td>
Double buffering + damage
</td>
</tr>
<tr>
<td>
Mouse / Keyboard
</td>
<td>
동작
</td>
<td>
현재 PS/2 polling
</td>
</tr>
<tr>
<td>
Window Manager
</td>
<td>
완료
</td>
<td>
drag/focus/z-order/min/max/close
</td>
</tr>
<tr>
<td>
Terminal / Shell
</td>
<td>
완료
</td>
<td>
기본 shell command 구현
</td>
</tr>
<tr>
<td>
Files
</td>
<td>
Placeholder
</td>
<td>
실제 filesystem 미연결
</td>
</tr>
<tr>
<td>
Settings
</td>
<td>
Placeholder
</td>
<td>
향후 구현
</td>
</tr>
<tr>
<td>
Timer / Interrupt
</td>
<td>
미구현
</td>
<td>
다음 기반 작업 후보
</td>
</tr>
<tr>
<td>
Memory Manager / Heap
</td>
<td>
미구현
</td>
<td>
향후 구현
</td>
</tr>
<tr>
<td>
VFS / Filesystem
</td>
<td>
미구현
</td>
<td>
Files 앱과 연결 예정
</td>
</tr>
</table>

# 다음 단계 후보

1. IDT 및 CPU exception handler
2. System timer / 실제 uptime
3. PS/2 keyboard/mouse IRQ1/IRQ12 event 기반 전환
4. Memory manager / heap
5. VFS 및 filesystem
6. Files 앱을 실제 filesystem과 연결
7. Settings 기능 구현
8. 이후 scheduler/process/application model 확장

# 주요 프로젝트 경로

```txt
kernel/src/framebuffer.rs
kernel/src/desktop.rs
kernel/src/input.rs
kernel/src/keyboard.rs
kernel/src/terminal.rs
kernel/src/shell.rs
kernel/src/platform.rs
kernel/src/main.rs
docs/architecture.md
README.md
```

# 현재 의미

현재 GyuOS는 Windows나 Linux 위에서 실행되는 GUI 프로그램이 아니다.

**UEFI → 자체 GyuOS Kernel → 자체 framebuffer renderer → 자체 Window Manager → 자체 Terminal** 경로로 VMware에서 직접 부팅되는 단계까지 진행됐다.

초기 목표인 “Codex를 이용해 자체 데스크톱 OS를 어디까지 만들 수 있는가”를 계속 실험한다.

