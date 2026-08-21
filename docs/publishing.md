# 글 작성과 발행

## 새 글 시작

기술 글은 먼저 `_drafts`에 작성한다.

```bash
cp docs/post-template.md _drafts/<slug>.md
bundle exec jekyll serve --drafts --livereload
```

공개 검토가 끝나면 날짜를 붙여 `_posts`로 옮긴다.

```bash
mv _drafts/<slug>.md _posts/YYYY-MM-DD-<slug>.md
```

## 발행

글마다 브랜치와 Pull Request를 하나씩 사용한다.

```bash
git switch -c post/<slug>
git add _posts/YYYY-MM-DD-<slug>.md
git commit -m "post: <title>"
git push -u origin post/<slug>
gh pr create --fill
```

Pull Request에서 공개정보 점검과 빌드 결과를 확인한 후 `main`에 병합한다. 병합된 변경은 GitHub Pages workflow가 자동으로 배포한다.
