# doublebump-legal

DoubleBump 앱의 App Store 심사용 법적 문서 사이트. GitHub Pages로 호스팅되는 정적 HTML 사이트.

## 구조

```
doublebump-legal/
├── index.html    → privacy.html로 자동 리다이렉트
├── privacy.html  → Privacy Policy 페이지
└── terms.html    → Terms of Service 페이지
```

## 배포

- Repository: https://github.com/macboii/doublebump-legal
- GitHub Pages로 호스팅 (Settings → Pages → Deploy from branch: main / root)
- Privacy Policy URL: https://macboii.github.io/doublebump-legal/privacy
- Terms of Service URL: https://macboii.github.io/doublebump-legal/terms

## 업데이트 방법

파일 수정 후 main 브랜치에 push하면 자동으로 반영됨.

```bash
git add .
git commit -m "Update legal documents"
git push origin main
```
