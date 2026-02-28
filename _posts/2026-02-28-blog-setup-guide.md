---
title: "GitHub Pages + Chirpy 블로그 초기 세팅 가이드"
date: 2026-02-28
categories: [블로그]
tags: [GitHub Pages, Jekyll, Chirpy, 세팅, macOS]
---

# GitHub Pages + Chirpy 블로그 초기 세팅 가이드

이 블로그는 **GitHub Pages**에 **Jekyll 테마 Chirpy**를 사용해 구성했습니다. macOS 기준으로 프로젝트 구조와 초기 세팅 방법을 정리합니다.

---

## 1. 개요

| 항목 | 내용 |
|------|------|
| **호스팅** | GitHub Pages |
| **엔진** | Jekyll (정적 사이트 생성) |
| **테마** | [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) |
| **배포** | GitHub Actions로 `main`/`master` 푸시 시 자동 빌드·배포 |
| **환경** | macOS (로컬 미리보기: Ruby + Bundler) |

---

## 2. 사전 요구사항

- **Git** 설치
- **Ruby** (Jekyll 실행용), **Bundler**, **Jekyll**
- **GitHub 계정** 및 개인/조직 레포지토리 (Pages 사용 가능)

아래는 **Mac에서 Chirpy 사용 기준**으로 진행한 환경 세팅 순서입니다.

---

### Mac에서 Ruby & Jekyll 세팅 (Chirpy 기준)

#### 1) Ruby 설치 확인

Mac에는 Ruby가 기본 설치돼 있지만 버전을 먼저 확인합니다.

```bash
ruby -v
```

없거나 구버전이면 Homebrew로 설치합니다.

```bash
brew install ruby
```

#### 2) Jekyll 설치

Bundler와 Jekyll을 gem으로 설치합니다.

```bash
gem install bundler jekyll
```

설치 확인:

```bash
jekyll -v
```

이후 프로젝트 폴더에서 `bundle install`로 Chirpy(Gemfile) 의존성을 설치하고, `bundle exec jekyll s -l`로 로컬 미리보기를 띄우면 됩니다.

---

#### 권한 오류가 났을 때: rbenv 사용 (강력 추천)

`gem install` 시 아래처럼 **권한 오류**가 나는 경우가 있습니다.

```
ERROR: While executing gem ... (Gem::FilePermissionError)
You don't have write permissions for the /Library/Ruby/Gems/2.6.0 directory.
```

macOS 기본 Ruby는 시스템이 쓰는 디렉터리라 **건드리지 않는 게 좋고**, 개발용 Ruby는 **rbenv로 따로 관리**하는 게 안전합니다. rbenv를 쓰면 gem을 사용자 디렉터리에 설치해서 권한 에러가 나지 않습니다.

**① rbenv 설치**

```bash
brew install rbenv
```

**② 쉘 설정 (zsh 기준)**

```bash
echo 'eval "$(rbenv init -)"' >> ~/.zshrc
source ~/.zshrc
```

**③ Ruby 설치 및 기본 버전 지정**

```bash
rbenv install 3.2.2
rbenv global 3.2.2
```

**④ Bundler 설치**

```bash
gem install bundler
```

이후 `gem install jekyll` 등 필요한 gem을 설치하면 **권한 에러 없이** 설치됩니다. 프로젝트 폴더에서 `bundle install` → `bundle exec jekyll s -l` 로 로컬 실행하면 됩니다.

---

## 3. 프로젝트 생성 방법

이 블로그는 **저장소에서 clone** 하는 방식으로 만들었습니다.

1. GitHub에서 [chirpy-starter](https://github.com/cotes2020/chirpy-starter) 또는 Chirpy 템플릿 저장소로 이동합니다.
2. **Use this template** 또는 fork로 본인 계정에 복사한 뒤, 원하면 저장소 이름을 `<username>.github.io`로 둡니다. (`https://<username>.github.io` 에서 접속 가능)
3. 로컬에서 clone 합니다.

   ```bash
   git clone https://github.com/<본인계정>/<저장소이름>.git
   cd <저장소이름>
   ```

4. clone 한 폴더 안에서 `bundle install` 로 의존성을 설치하고, `_config.yml` 에서 title, url, github username, social 등만 본인 정보로 수정합니다.
5. `bundle exec jekyll s -l` 로 로컬에서 미리보기 확인 후, push 하면 GitHub Actions가 빌드·배포합니다.

정리하면, **템플릿 저장소 → 본인 레포로 복사 → clone → 설정 수정** 순서로 진행하면 됩니다.

---

## 4. 프로젝트 디렉터리 구조

```
레포지토리 루트/
├── .github/
│   └── workflows/
│       └── pages-deploy.yml   # GitHub Actions: 빌드 & Pages 배포
├── _config.yml                # 사이트 전역 설정 (제목, URL, 테마, 메뉴 등)
├── _data/
│   ├── contact.yml            # 사이드바 연락처 링크 (GitHub, 이메일 등)
│   └── share.yml              # SNS 공유 설정
├── _plugins/
│   └── posts-lastmod-hook.rb  # 글별 "마지막 수정일" 자동 반영 (git 기반)
├── _posts/                     # 블로그 글 (마크다운)
│   └── YYYY-MM-DD-제목.md
├── _tabs/                      # 상단 네비게이션 메뉴 (페이지 목록)
│   ├── about.md
│   ├── archives.md
│   ├── categories.md
│   └── tags.md
├── index.html                  # 메인 페이지 (layout: home)
├── .nojekyll                   # GitHub Pages가 Jekyll 빌드를 건너뛸 때 필요 (Actions 사용 시)
├── Gemfile                     # Ruby 의존성 (jekyll-theme-chirpy 등)
└── tools/
    ├── run.sh                  # 로컬 서버 실행 스크립트 (bundle exec jekyll s -l)
    └── test.sh                 # 빌드/테스트
```

- **글**: 모두 `_posts/` 아래에 `YYYY-MM-DD-슬러그.md` 형식으로 두고, 상단에 front matter로 제목·날짜·카테고리·태그를 지정합니다.
- **메뉴**: `_tabs/` 안의 마크다운 파일이 상단 탭이 됩니다. `order` 값으로 순서를 정합니다.
- **배포**: 실제 배포는 GitHub Actions가 수행하고, 빌드 결과만 `_site`에 생성한 뒤 GitHub Pages에 올립니다. `.nojekyll`이 있으면 Jekyll의 기본 빌드 대신 Actions 결과를 그대로 서빙할 수 있습니다.

---

## 5. 필수 설정: \_config.yml

사이트 제목, URL, 소셜 링크 등은 `_config.yml`에서 한 곳만 바꾸면 됩니다.

### 5.1 사이트 기본 정보

```yaml
title: ITLJY                    # 사이트 제목
tagline: Developer Blog         # 부제목
description: >-                  # SEO·피드용 설명
  Developer Blog by Lee Ji Yeon

# GitHub Pages 루트에 배포하면 보통 비워 둠. 서브 경로면 예: "/blog"
baseurl: ""
url: ""                          # 배포 후 예: "https://username.github.io"
```

### 5.2 GitHub / 소셜

```yaml
github:
  username: ITLJY

social:
  name: Lee Ji Yeon
  email: layeon@naver.com
  links:
    - https://github.com/ITLJY
```

### 5.3 테마·기능

- **theme**: `jekyll-theme-chirpy` (Gemfile에 해당 gem 필요)
- **lang**: `en` 또는 `ko` 등
- **theme_mode**: `light` / `dark` 또는 비워 두면 시스템 설정 따름
- **toc**: `true` — 글 내 목차(Table of Contents) 사용
- **comments**: 댓글 사용 시 `disqus` / `utterances` / `giscus` 중 하나 설정
- **analytics**: Google Analytics 등 원하면 ID 입력

### 5.4 글·카테고리·태그 (jekyll-archives)

```yaml
jekyll-archives:
  enabled: [categories, tags]
  layouts:
    category: category
    tag: tag
  permalinks:
    tag: /tags/:name/
    category: /categories/:name/
```

글의 front matter에 `categories: [Ruxpress Project]`, `tags: [Spring Boot]`처럼 넣으면 해당 카테고리/태그 페이지가 자동 생성됩니다. **카테고리는 폴더가 아니라 글에 지정한 값으로만 생성됩니다.**

---

## 6. 로컬에서 미리보기

```bash
# 의존성 설치 (최초 1회)
bundle install

# 로컬 서버 실행 (Live Reload 권장)
bundle exec jekyll s -l
# 또는
bash tools/run.sh
```

브라우저에서 `http://127.0.0.1:4000` (또는 터미널에 표시된 주소)로 접속해 확인합니다. `_config.yml`이나 글을 수정하면 자동으로 갱신됩니다.

---

## 7. GitHub Pages 배포 (GitHub Actions)

이 프로젝트는 **GitHub Actions**로 빌드·배포합니다. Jekyll을 GitHub에서 직접 돌리는 방식이 아닙니다.

### 7.1 워크플로우 동작

- **파일**: `.github/workflows/pages-deploy.yml`
- **트리거**: `main` 또는 `master` 브랜치에 push (단, `.gitignore`, `README.md`, `LICENSE` 변경만 있으면 제외 가능하도록 설정 가능)
- **순서**:
  1. Ruby 3.3, Bundler 캐시 설정
  2. `bundle exec jekyll b -d "_site"` 로 정적 사이트 빌드
  3. `htmlproofer`로 내부 링크 등 간단 검증
  4. 빌드 결과를 GitHub Pages 아티팩트로 업로드
  5. **Deploy to GitHub Pages** 액션으로 실제 배포

### 7.2 저장소 설정

1. GitHub 저장소 **Settings → Pages**
2. **Source**: **GitHub Actions** 선택
3. `main`(또는 `master`)에 push하면 자동으로 빌드 후 `https://<username>.github.io`(또는 설정한 도메인)에 반영됩니다.

---

## 8. 글 작성 방법

새 글은 `_posts/` 아래에 다음 형식으로 만듭니다.

**파일명**: `YYYY-MM-DD-제목-슬러그.md`  
(예: `2026-02-28-blog-setup-guide.md`)

**front matter 예시**:

```yaml
---
title: "글 제목"
date: 2026-02-28
categories: [블로그]      # 하나 또는 여러 개
tags: [Jekyll, GitHub Pages]
---
```

이후 본문은 일반 마크다운으로 작성합니다. Chirpy는 기본적으로 `permalink: /posts/:title/` 형태로 글 URL을 만들고, 카테고리·태그는 위 `jekyll-archives` 설정대로 `/categories/이름/`, `/tags/이름/`으로 생성됩니다.

---

## 9. 정리

| 단계 | 내용 |
|------|------|
| 1 | Chirpy Starter 템플릿 사용 또는 기존 레포에 `jekyll-theme-chirpy` Gem 추가 |
| 2 | `_config.yml`에서 title, url, github, social 등 수정 |
| 3 | `_tabs/`, `_data/contact.yml` 등으로 메뉴·연락처 설정 |
| 4 | `bundle install` 후 `bundle exec jekyll s -l` 로 로컬 확인 |
| 5 | `.github/workflows/pages-deploy.yml` 로 push 시 자동 빌드·배포 |
| 6 | 저장소 Settings → Pages → Source: GitHub Actions 선택 |

이 순서대로 진행하면 macOS에서 GitHub Pages + Chirpy 기반 기술 블로그 초기 세팅을 마칠 수 있습니다. 세부 옵션(댓글, 분석, PWA 등)은 Chirpy 공식 문서를 참고해 필요한 것만 켜면 됩니다.
