# LYJ0304 기술 블로그

공개 주소는 [lyj0304.github.io/blog](https://lyj0304.github.io/blog/)입니다.

## 로컬 실행

```sh
nvm use
npm ci
npm run dev
```

Node.js 24를 사용합니다. 브라우저에서 개발 서버의 `/blog/` 경로를 열면 됩니다. 로컬 빌드는 `npm run build`로 확인할 수 있습니다.

## 글과 배포

글은 `src/content/blog/`에서 Markdown 또는 MDX로 관리합니다. 글 상세 페이지의 공개 경로는 `/blog/posts/<slug>/`입니다.

`main`을 대상으로 PR을 열면 GitHub Actions에서 빌드를 검사합니다. `main`에 변경 사항이 들어오면 빌드 후 GitHub Pages로 배포합니다.
