# 성승재 기술 블로그

GitHub Pages와 Jekyll Chirpy 테마로 만든 기술 블로그입니다.

## 로컬 실행

Ruby와 Bundler가 설치되어 있다면 아래 명령으로 확인할 수 있습니다.

```bash
bundle install
bundle exec jekyll serve
```

## 글 작성

글은 `_posts` 폴더에 `YYYY-MM-DD-title.md` 형식으로 추가합니다.

## 배포

GitHub Pages 설정에서 Source를 `GitHub Actions`로 선택하면 `.github/workflows/pages-deploy.yml`을 통해 배포됩니다.
