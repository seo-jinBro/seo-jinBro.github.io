# Blog

seo-jinBro's Dev Blog

## How to use

우선, rbenv 설치

```bash
# install rbenv
brew install rbenv ruby-build
rbenv install 2.6.8
rbenv global 2.6.8

# 아래 라인을 .zshrc 에 추가
eval $(rbenv init - zsh)

# install bundler
gem install bundler

# install packages
bundle

# Syntax highlighting 문제로 아래를 추가
gem install kramdown rouge --user-install
```

그 다음, 직접 실행해보려면 아래 실행

```bash
# build
bundle exec jekyll build --watch --incremental

# run
bundle exec jekyll serve
```

## Google Analytics

템플릿이 상속받아서 쓰는 구조라서, `analytics.html` 안에 해당 설정이 되어있음.
따라서 `default.html` 에서 이를 include 하는 식으로 처리하는데, 문제는 production 일때만 로드하게 되어있음.

개발환경에서 테스트해보려면 아래와 같이 테스트 가능

```bash
JEKYLL_ENV=production bundle exec jekyll serve
```

## Reference

- [minimal-mistakes-jekyll-theme](https://mademistakes.com/work/minimal-mistakes-jekyll-theme/)
