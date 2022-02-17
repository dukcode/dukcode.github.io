---
layout: single
title: "[jekyll] Mathjax 오류, Mathjax3 설치로 해결하는 법"
categories: tools
tag: [jekyll, mathjax, tip]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---
## 개요
> **기존 mathjax의 오류로 수식 렌더링이 적용 안됐던 부분을 mathjax3 설치로 해결한다.**

15개 이상의 블로그에 나온 **mathjax** 적용법을 모두 적용해 보았지만 `$ $`의 인라인 모드는 잘 동작하지만 `$$ $$`의 인클로즈 모드의 결과가 인라인 모드와 같이 뜨거나 `\( \)`에 둘러 쌓여 수식 적용이 안되는 오류가 생겼다.  

긴 서치 끝에 다음 방법으로 오류를 해결할 수 있었다.

## 방법

* _config.yml을 열어 다음과 같이 엔진이 `kramdown`인지 확인한다.
```yml
markdown: kramdown
```  

* `/_includes/` 폴더에 `mathjax.html`을 생성하고 다음과 같이 입력하고 저장한다.
```html
<script>
MathJax = {
  tex: {
    inlineMath: [ ['$', '$'], ['\\(', '\\)'] ]
  },
  svg: {
    fontCache: 'global'
  }
};
</script>
<script
  type="text/javascript" id="MathJax-script" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js">
</script>
```

{% raw %}
* `/_includes/` 폴더의 `head.html`을 열어 다음과 같이 입력하고 저장한다.\
  ```html
    {% include mathjax.html %}
  ```
{% endraw %}

* github에 push해보면 다음과 같이 mathjax3가 잘 적용됨을 확인 할 수 있다.

> 알고리즘의 시간복잡도는 `$O(n^2)$`이다.\
> 알고리즘의 시간복잡도는 다음과 같다.
> `$$O(n^2)$$`

> 알고리즘의 시간복잡도는 $O(n^2)$이다.\
> 알고리즘의 시간복잡도는 다음과 같다.
> $$O(n^2)$$
