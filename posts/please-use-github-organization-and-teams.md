---
title: 조직에서 GitHub Developer Plan을 사용하는 것은 안티패턴이다.
author: Hyeseong Kim
date: 2018-09-19
tags:
  - github
  - collaboration
references:
  - https://github.com/pricing
  - https://help.github.com/articles/about-teams
  - https://help.github.com/articles/converting-a-user-into-an-organization
---

> TR;DL - 돈 아깝다고 GitHub 한 계정 돌려쓰기 하지 마세요.

이 글은 내가 속한 회사/앞으로 속하게 될 회사에 불평불만/제안 하는 것을 목적으로 두고있다.

# 돈 아끼기?

예전에 GitHub Enterprise 솔루션 지원하던 때, GitHub 사용자 계정을 프로젝트 그룹처럼 활용하는 경우를 종종 본 적이 있었다.

안타깝게도 현재 회사에서 사용하는 방식이기도하다.

- GitHub Account @user가 있다.
- @user 계정은 Developer Plan을 구독하고 있다.
- @user 계정으로 Private Repository를 생성해서 내부 프로젝트를 관리한다.
- 프로젝트에 Involve 된 직원의 계정을 Collarborator로 등록한다.
- 프로젝트 관리는 한명의 매니저가 담당하고 있다.

[GitHub 가격정책](https://github.com/pricing)을 보면 Annual로 유저당 9달러(5명까지는 5달러)를 내는 Team Plan에 비해서 Developer Plan이 압도적으로 싸다. Team Plan을 선택하면 회사에 개발자가 20명만 되도 한 달에 1,920달러가 나간다. 

그런데 기능표로 보기엔 별 차이가 없어보이니 적절한 선택으로 보인다. 이정도면 훌륭한 원가절감 아닌가?

# Organization 기능 들여다보기

하지만 사실 GitHub을 자주 사용할 수록, 기능을 알게 될 수록 의문점이 생긴다.

Developer / Team Plan은 표에 나온 것 만큼 기능 차이가 별로 없을까? 기능 차이가 없다면 가격은 왜 이렇게 차이날까? 표에서는 **Team permissions**, **Organization permissions** 딱 두 항목만이 차이날 뿐이다.

한 번이라도 Organization에 속한적이 있다면 알게된다. 혼자 하는 프로젝트를 제외한 GitHub의 모든 Use Case가 Organization에 녹아 있다는 것을. 어떤 서비스던지 제시된 Use Case를 역행하면 불편해지지 않을 수가 없다.

## Team, Team, Team!

레파지토리 목록만 본다면, 사용자 계정과 무슨 차이가 있나 싶다.

사실 별 것 없어보이는 Organization의 핵심은 Teams 탭에 있다. Team은 Organization 멤버의 그룹이다.

### 팀 단위 협업

일단 처음 Team을 생성해보면 이렇게 팀원들끼리 협업할 수 있는 공간이 생긴다.

![GitHub Team Page](https://help.github.com/assets/images/help/organizations/team-page-discussions-tab.png)

GitHub 가이드에서 가져온 이 오래된 스크린샷과는 다르게, Projects Board 기능 또한 팀 단위로 사용할 수 있다.

조직의 Jira, Conflence, Trello 등 협업/이슈트래커 도구의 활용도가 낮다면, 완전히 대체, 통합해버릴 수도 있는 훌륭한 기능이다.

### 팀 별 권한 관리

개인 레파지토리 경우 사용자를 추가하고 싶다면 Collaborators로 추가할 수 있고, Collaborator는 master 브랜치에 Push/Merge 하는 권한만 공유받게 된다.

즉, 여전히 실질적으로 프로젝트를 "관리"할 수 있는 건 @user 계정으로 접속하는 매니져 한 사람 뿐이다.

반면, Organization의 레파지토리는 팀 단위로 사용자에게 "관리" 권한을 포함한 세부적인 권한을 부여할 수 있다.

![GitHub Team Permissions](https://help.github.com/assets/images/help/organizations/team-repositories-change-permission-level.png)

이건 장단점의 영역이라기보단 필수요소라 볼 수 있겠다.

- 그룹단위로 관리할 수 있다는 건 매니저에게 없어서 안될 기능이다.
- 권한의 세부적으로 컨트롤 또한 기업의 소스 컨트롤이라면 필수이다.

특히 신규 멤버가 추가되거나, 퇴사자가 발생하는 경우 더욱 빛을 발한다.

## API 접근 제어

GitHub는 다양한 도구들과 통합(Integration)되었을 때 그 전체를 활용할 수 있는 도구라는 점에서 API의 접근 제어가 세분화되는 것도 중요한 요소로 볼 수 있다.

@user 계정을 사용하고자 한다면 매니져가 OAuth Authorize 하는 과정을 전부 직접 수행해야 한다.

특정 프로젝트에 특화된 도구를 연동하는데, 그 프로젝트의 수행자는 레파지토리의 "Settings" 메뉴에 접근조차 할 수 없고, 결국은 매니저가 사용되는 모든 도구를 숙달하고 있어야 한다는 의미이다.

CI/CD(Continuous Integration/Deployment)라면 Access Key의 적절한 관리가 프로젝트 라이프사이클에 직접적인 영향을 주기때문에 더더욱 중요해진다.

이 모든 걸 프로젝트 수행자가 적절한 권한을 부여받지 못해 손가락 빨며 기다려야 하는가? 그리고 매니저는 이걸 다 직접 일일히 관리하고 있을 시간이 있는가?

## 조직 구성 관리

GitHub로 조직의 Hierarchy를 관리할 수 있다는 건 더 많은 가능성을 파생시킨다.

소규모 조직 중에 LDAP/SAML로 SSO 구축해서 쓰는 곳이 얼마나 있을까? (있다면 벌써 Enterprise Plan을 고려하고 있었겠지..) 그렇기에 작은 조직일 수록 사용자의 계정을 주먹구구식으로 관리하는 경향이 크다.

이런 상황에서 어느 한 군데라도 LDAP 비스무리하게 조직 구성 관리가 수행되고 있다는 것은 개발용 서비스/도구에서의 Access Controll에 활용될 가능성을 내포하고 있다. (인하우스 도구 만들 때 이 구성관리의 유무가 엄청난 차이를 만든다.)

그리고 GitHub API는 OAuth 2.0 Service Provider를 지원하고 있기도 하다. 개발도구로 제한한다면 Google 계정이 통합되는 서비스만큼이나 통합되는 서비스의 범위가 넓고, 이 중에서 실제로 Organization 정보를 활용하는 협업도구도 종종 볼 수 있다.

## 알림 제어

협업 도구가 가장 중요하게 다루는 기능이 바로 **"알림(Notification)"**이다. 잘 제어되는 알림은 비동기 협업의 가장 핵심적인 부분이라고 할 수 있다.

이 부분 역시 계정을 돌려쓰다보면 Organization의 부재로 많은 것을 잃어버리게 된다.

### 그룹 멘션

알림기능에서 대상을 구분하는 용도로 멘션 기능을 활용하게 된다. @user1, @user2 라는 식으로 본문 내용에 포함하면 이 내용은 @user1, @user2 에게 더 높은 중요도를 가지게 된다.

Organization, Team이 있다면 이 단위로 멘션이 가능하다. 멘션을 **"역할"** 단위로 구분할 수 있는 것이다.

멘션은 서비스 통합 시에도 자주 활용되는데, Organization, Team 단위는 컨텍스트로 다루기도 좋다.

### 알림 커스텀 라우팅

이 기능 못써서 심각하게 불편해하고 있다.

나는 GitHub에서 회사 프로젝트 뿐 아니라 오픈소스 프로젝트의 이슈들도 다수 Watch하고 있어서 GitHub에서 오는 알림 메일이 개인 Inbox의 대다수를 차지한다.

물론 회사용 메일은 따로 있지만 GitHub 계정의 Primary Email은 개인 메일이라 분리가 안된다.

GitHub는 이러한 상황을 위해 Personal Setttings > Notifications > Custom Routing 메뉴에서 **"특정 Organazation에서 오는 알림을 다른 메일 주소로 보내는 기능"**을 제공하는데, 이걸 못쓰니 회사 프로젝트의 GitHub 알림은 거의 놓치고 있다. (애초에 회사에서 메일 클라이언트 따로 쓰고 있지도 않아서, 스마트폰으로 확인한다 -_-)

이 것 때문에 "회사용 GitHub 계정"을 따로 파는 분도 있는 것 같다..;

## 그래서..

혼자 하는 프로젝트가 아닌 이상, Organization 하나로 GitHub의 활용도 차이가 크다. 사실, "협업"을 주제로 하는 모든 서비스가 다 그렇다.

가격 때문에 Developer Plan을 선택하고, 여기서 발생하는 제약사항 때문에

- 매니저가 무수히 많은 책임/역할에 묶이고,
- 막상 프로젝트의 수행자는 적절한 권한을 못 받고,
- 기능제약으로 생긴 불편함에 대한 책임을 개인에게 전가하고,
- 정돈된 리소스로부터 파생될 다양한 통합가능성을 배제하는,

이러한 행태들이 내가 보기엔 그리 적절하지 못한 것 같다.

얘기 꺼내보기 전에는 다른 팀원들이 어떻게 생각하고 있을지는 모르겠다. 만약, [불편함이 존재하는 것을 인지하지 못하는 상태](https://markhneedham.com/blog/2011/01/26/the-five-orders-of-ignorance-phillip-g-armour)라면 개인적으로 설득할 자신도 없고, 정말 안타까워할 것 같다.

# 가격 합리성 따지기

기능 제약이 많다는걸 이해한 후에도 가격이 너무 부담된다면,

직접 구축해서 사용할 수 있는 오픈소스 서비스로 [GitLab CE](https://gitlab.com/gitlab-org/gitlab-ce), [Gogs](https://gogs.io), [Yona](https://repo.yona.io) 등 다양한 옵션이 있다. (사실 GitLab이 순기능만 따지면 훨씬 낫..)

하지만 전 회사에서 GitLab CE를 직접 구축해서 써본 경험으로는 이게 좋은 대안이라고 내세우고 싶지 않다. 구축/운영비용이 만만치 않고, 조직 내 의존도가 아주 높은 서비스임에도 불구하고 구축/운영비용이 눈에 잘 보이지 않아 무시되었던 경험이 있기 때문이다.

구매에 실패하지 않는 법으로 "가장 많은 시간을 들이는 것에 많은 돈을 써라" 라는 내용을 트위터에서 본적이 있다. (원문을 아시는 분은 댓글 좀 부탁드리겠습니다.)

GitHub는 현재 내 개인/회사 통틀어서 단언컨데 거의 대부분의 시간을 함께하는 서비스이다. 깃헙에 들이는 시간, 기능 제약으로 인한 생산성 저하, 여기서 나오는 시간을 임금으로 환산할 수 있다면 개발자당 월 9달러가 그렇게 큰 비용일까? (정확한 값은 어떻게 측정될 지 모르겠으나, 예/아니요 논리를 따지는 건 개발자 임금 수준을 생각하면 상당히 예측하기 쉬운 문제로 보인다.)

만약 Youtube 보는게 일이였다면 ~~좋겠다~~ 계산이 쉬워졌겠다. 유튜브 광고보고 있는데 들인 시간(한시간에 5분이라 치자)이랑 하루 노동시간 따졌을 때,  (최저임금 8,350원 x 5분/1시간 x 한달 근로시간 209시간) = 145,429원 나오는데 여기에 러프하게 반토막해놓고 따져도 회사에서 Youtube Premium (한달 7,900원) 사주는게 손해겠냐는 얘기다.

그리고 나는 GitHub를 통해 소스 관리하고 협업하는게 YouTube 쳐다보고 있는 것보단 훨씬 고부가가치 작업이라고 믿는다.

# 결론

> ~~ 사주세요~~
> 결국 제 값인 데는 다 이유가 있다.

라고 말하고 싶다. 특히 내가 속한/속하게 될 조직은 지속적인 생산성과 직결된 부분엔 되도록 돈을 아끼지 않았으면 좋겠다.
