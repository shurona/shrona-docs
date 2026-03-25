---
tags:
  - electron
---
# 학습용 프로토타입 박치기 해보기
electron 사용을 위해서 설치를 진행해본다. 
## 처음 설치 진행
```shell
npm init -y

npm install electron --save-dev
```

# 일단 띄워보자
## Electron의 핵심 구조 (3개 파일)

### 1. `main.js` - Main Process (백엔드)
- Node.js 환경에서 실행
- 앱 생명주기 관리 (시작, 종료)
- 브라우저 창(BrowserWindow) 생성
- 파일시스템, DB, OS API 등 접근 가능
- **Spring의 Application.java + Controller 역할**

### 2. `preload.js` - 다리 역할
- Main Process와 Renderer Process를 연결
- Renderer에서 사용할 수 있는 API를 안전하게 노출
- **Spring의 DTO/API 인터페이스 역할**

### 3. `index.html` - Renderer Process (프론트엔드)
- Chromium 브라우저에서 실행
- HTML/CSS/JS로 UI 구성
- 직접 Node.js API 사용 불가 (보안상 preload를 통해 접근)
- **일반 웹페이지와 동일**

## 통신 흐름
```
[Renderer]  ──IPC 요청──▶  [preload.js]  ──▶  [Main Process]
(버튼 클릭)                  (다리 역할)        (파일 저장, DB 조회)
                                                     │
[Renderer]  ◀──IPC 응답──  [preload.js]  ◀──  [Main Process]
(결과 표시)                                    (결과 반환)
```
## 창 띄워보기
### main.js
```javascript
const { app, BrowserWindow} = require('electron');
const path = require('path');

function createWindow() {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })

  win.loadFile('index.html');
};

app.on('window-all-closed', () => {
  app.quit()
});
```
- app - 앱의 생명 주기를 관리한다.
- BrowserWindow - 브라우저의 창을 여는 것과 동일하다.
- `app.whenReady()` - 앱 준비되면 창 생성
### proload.js
```javascript
const { contextBridge } = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  // 나중에 여기에 Renderer가 사용할 함수들을 추가
  getAppVersion: () => process.versions.electron
})
```
- contextBridge.exposeInMainWorld - Renderer에 `window.electronApi`로 접근할 수 있도록 한다.
- 보안을 위해 필요한 기능만 접근하도록 하는 것
# electron에서 통신 방법
## IPC란
Inter-Process Communication - Main Process와 Renderer Process 간 통신 방법
- Renderer → Main = 프론트엔드에서 API 호출 (fetch/axios)
- Main → Renderer = API 응답 반환
