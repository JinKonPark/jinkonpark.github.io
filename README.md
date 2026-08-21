# JinKonPark의 기술 기록

AI 에이전트와 LLM 시스템을 만들며 배운 설계, 평가, 운영 원칙을 기록하는 GitHub Pages 블로그다.

- 사이트: <https://jinkonpark.github.io>
- 글 작성 절차: [`docs/publishing.md`](docs/publishing.md)
- 글 템플릿: [`docs/post-template.md`](docs/post-template.md)

## 로컬 확인

Ruby 3.4와 Bundler를 준비한 뒤 실행한다.

```bash
bundle install
bundle exec jekyll serve --drafts --livereload
```

프로덕션 빌드와 내부 링크 검사는 다음 명령으로 실행한다.

```bash
bash tools/test.sh
```

테마는 [Jekyll Theme Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)를 사용한다.
