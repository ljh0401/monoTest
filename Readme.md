## monorepo 가능성 탐색

- 해당 방식을 따라할 경우 Node.js가 install 되어있어야 합니다.
- 추가적으로 환경 변수를 통해 cmd.exe를 찾아갈 수 있어야 합니다. ( Error: spawn cmd.exe ENOENT)
    - 만약 Error 발생 시 C:\Windows\System32\cmd.exe이 존재하는지 확인
    - cmd에서 `echo %PATH%` 를 통해 "C:\Windows\System32" 가 출력 되었는지 확인
    - 없을 경우 "시스템 환경 변수 편집" -> 환경 변수 ->  "시스템 변수"의 "새로 만들기"로 "C:\Windows\System32" 경로 추가
    - 이후 관리자 권한으로 실행한 cmd에서 `setx PATH "%PATH%;C:\Windows\System32"` 실행
    - 컴퓨터 재부팅


#### 시작
1. 우선 nestjs를 쓸 예정이니 nestjs/cli를 npm을 통해 설치합니다.
```
    npm i -g @nestjs/cli
```
2. 그 후 monorepo로 만들 폴더를 하나 만들고, 거기로 이동합시다. (전 monoTest로 만들었습니다)
```
    mkdir monoTest && cd monoTest
```
3. 그 다음에는 repo들을 담을 apps를 하나 만들어봅시다.
```
    mkdir apps && cd apps
```

4. 그 곳에 next, nest를 만들겠습니다.
```
    nest new backend --strict --skip-git --package-manager=npm
    npx create-next-app@latest frontend --typescript
```
--strict는 TypeScript strict mode option을 추가해주어, error를 좀 더 빨리 잡을 수 있도록 도와준다고 합니다.

5. 동시 실행 시 로그를 좀 더 잘 보고자, 그리고 병렬 실행을 보장하기 위해 "concurrently"을 사용할 것입니다.
```
npm i concurrently
```
6. 그리고 package.json에 다음과 같이 작성 해주세요
```
{
  "private": true,
  "workspaces": [
    "apps/backend",
    "apps/frontend"
  ],
  "scripts": {
    "dev": "concurrently \"npm:dev:backend\" \"npm:dev:frontend\"",
    "dev:backend": "cd apps/backend && npm run start",
    "dev:frontend": "cd apps/frontend && npm run dev",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "concurrently": "^9.1.2"
  }
}
```
private는 배포 시 repo 관리 부분은 배포를 하지 않기 위해 true를,
workspaces는 관리하는 repo들을, scripts는 상위 폴더에서 npm run dev 시 concurrently로 둘을 실행시키기 위한 script를 작성합니다.

7. 이후 npm install 후 npm run dev를 monoTest에서 실행해주면 monorepo 생성 끝!
참고로
```
npm warn deprecated inflight@1.0.6: This module is not supported, and leaks memory. Do not use it. Check out lru-cache if you want a good and tested way to coalesce async requests by a key value, which is much more comprehensive and powerful.
npm warn deprecated glob@7.2.3: Glob versions prior to v9 are no longer supported
npm warn deprecated glob@7.2.3: Glob versions prior to v9 are no longer supported
```
이 부분은 nest에서 사용하는 module 관련 문제입니다다. (직접 업데이트하면 해결은 가능하지만, 저는 따로 건드리지 않았습니다.)


#### 추가적으로 설정 or 고민해야하는 부분

1. 공유 패키지 구성, 설정 파일 구성
      -  모노레포의 장점인 코드 재사용성을 위해서는 공유 패키지를 구성해야 합니다.
      -  또한 ESlint, Prettier 등의 기본 설정을 root에 파일로 만들어 관리해야 합니다.
      - 이를 위해서는 디렉토리와 파일을 추가해야 하나, 가능성만 살펴본 현재 test에서는 따로 구성하지 않았습니다.

2. 패키지 메니저 변경
      - 현재는 npm을 사용하고 있으나, 제대로 사용을 할 예정이면 Yarn이나 Lerna, 더 나아가 NX, Turborepo를 사용하는 것이 좋아 보입니다.
      - 현재의 저는 단순히 공통 요소를 공유하는 것이 목적이기에, 다음 프로젝트에 monorepo 사용시 Yarn을 통한 구성을 해 볼 예정입니다. (관리를 편하게 해주는 Lerna 더 나아가 시각화를 제공하는 NX를 사용한다 하더라도 해당 기능을 활용하지 않을 것 같기 때문입니다.)

3.  node_modules 관리에 대한 고민
      - 현재 root의 node_modules에서 전체 패키지를 관리하고 있습니다.
      - 후에 프로젝트들이 쌓일수록 패키지가 늘어나 관리와 의존성 검색에 많은 시간이 소요될 것으로 예측됩니다.
      - 추가적으로 중복되어 설치되는 node_modules를 Hoisting해서 생긴다는 유령 의존성(Phantom Dependency) 문제로 패키지가 늘어날수록 더 큰 문제가 발생할 것이라 생각했습니다.
      - 이러한 부분은 yarn berry를 사용하면서 Plug'n'Play을 통해 검색과 의존성 문제를 해결할 수 있을 것으로 보입니다.
      - 추가적으로 zero install까지 가능하게 만든다면 의존성 관리를 좀 더 편리하게 가능해질 것으로 보입니다.
      - 이에 다음에 모노레포를 구성할 때는 yarn berry를 사용해 볼 것 같습니다.

4. back과 front를 한 repo에서 관리하는 것이 도움이 될 것인지에 대한 고민
      - 버전의 일관성을 유지하고 코드를 공유할 때에 모노레포는 도움이 되는 것이 맞으나, back과 front의 코드 공유가 많을 것인지에 대한 고민이 들었습니다.
      - 만약 코드의 공유가 많다면, 그리고 개발 단계에서 타입 등의 공통 부분의 지속적인 수정이 예측된다면 모노레포의 사용으로 얻을 수 있는 장점이 많을 것으로 보이나, 코드의 공유가 적고 배포 주기가 다르다면 오히려 통합된 의존성, 버전 관리가 단점으로 작용할 수 있을 것입니다.
      - 이에 현재 진행하는 프로젝트는 우선 멀티레포로 진행하고, 시간적 여유가 남을 경우 모노레포로 전환해보며 차이점에 대해 살펴볼 예정입니다.
