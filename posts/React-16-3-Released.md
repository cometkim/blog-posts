---
title: React 16.3 변경사항과 소식들
author: Hyeseong Kim
date: 
tags:
  - react
---

# 새로워진 Context API

# 라이프사이클 메서드 변경
3개의 componentWill... 라이프사이클 메서드의 사용이 중단된다.
- `componentWillMount`
- `componentWillUpdate`
- `componentWillRecieveProps`

대신 static 메서드가 하나 추가 `getDerivedStateFromProps`

# StrictMode

# AsyncMode