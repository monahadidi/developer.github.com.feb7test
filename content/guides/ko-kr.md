---
제목 : CI 서버 구축
---

# 지속적인 통합 서버 구축

{:toc}

는 [Status API] [status API] 함께 범 묶는에 대한 책임
있도록 테스트 서비스는 모든 사용자가 테스트를 나타낼 수 있도록 밀어
풀 요청한다. {{ site.data.variables.product.product_name }}

이 가이드는 당신이 사용할 수있는 설정을 보여 해당 API를 사용합니다.
우리의 시나리오에서, 우리는 것입니다 :

* 끌어 오기 요청을 열 때 우리의 CI 제품군을 실행 (우리는 보류에 CI 상태를 설정할 수 있습니다).
CI를가 완료 * 때, 우리는 그에 따라 끌어 오기 요청의 상태를 설정합니다.

우리의 CI 시스템과 호스트 서버는 우리의 상상의 허구 될 것입니다. 그들은 수
트래비스, 젠킨스, 또는 완전히 다른 것. 이 가이드의 핵심은 설정됩니다
과의 통신을 관리하는 서버를 구성.

당신이하지 않았다면, 반드시, 그리고 방법 [download ngrok] [ngrok]
에. [use it] [using ngrok] 우리는 지역의 노출에 대한 매우 유용한 도구로 발견
사이.

참고 :이 프로젝트의 전체 소스 코드를 다운로드 할 수 있습니다
[from the platform-samples repo][platform samples].

## 서버를 작성

우리는 우리의 로컬 연결이 작동하는지 증명하기 위해 빠른시나 응용 프로그램을 쓸 것이다.
이제이 시작하자 :

```루비
필요 '시나'
'JSON'을 필요로

이후 '/ event_handler'할
페이로드 = JSON.parse [:payload] (PARAMS)
"음, 일했다!"
종료
```

(당신이시나의 작동 방식에 익숙하지 않은 경우, 우리는 좋습니다.) [reading the Sinatra guide] [Sinatra]

이 서버 최대를 시작합니다. 당신이 원하는 것, 그래서 기본적으로,시나는 포트`4567`에서 시작
너무, 그것에 대해 청취를 시작 ngrok 구성 할 수 있습니다.

작업이 서버에 대한 위해, 우리는은 webhook와 저장소를 위로 설정해야합니다.
은 webhook는 끌어 오기 요청이 작성 또는 합병 될 때마다 화재 구성해야합니다.
가서 당신이 장난이 마음에 저장소를 작성. 할 수 있음 우리는
제안 [@octocat's Spoon/Knife repository] (https://github.com/octocat/Spoon-Knife)?
그 후, 당신은 그것에게 URL을 먹이 저장소에 새으로 webhook을 만듭니다
그 ngrok는 준과 같은 선택 '을 application / x-www-form-urlencoded`
콘텐츠 형식 :

! [A new ngrok URL] (/assets/images/webhook_sample_url.png)

** ** 업데이트은 webhook을 클릭합니다. 당신은`글쎄, 그것은했다!`의 신체 반응을 볼 수 있습니다.
큰! 클릭 ** 나 개​​별 이벤트 **을 선택하고 다음을 선택하자 :

* 상태
* 풀 요청

이러한 이벤트가 우리의 서버에 {{ site.data.variables.product.product_name }} 보낼 때마다 관련 조치
발생합니다. 의는 * 단지 * 지금 끌어 오기 요청 시나리오를 처리하기 위해 서버를 업데이트하자 :

```루비
이후 '/ event_handler'할
@payload = JSON.parse [:payload] (PARAMS)

케이스 request.env ['HTTP_X_GITHUB_EVENT']
때 "pull_request"
@payload는 == ["action"] "열"경우
Process_pull_request ["pull_request"] (@payload)
종료
종료
종료

헬퍼는 할
데프 process_pull_request (pull_request)
두고 "그것은 #입니다" {pull_request['title']}
종료
종료
```

무슨 일이야? 발송 모든 이벤트는 'X-GitHub의-Event`를 {{ site.data.variables.product.product_name }} 부착
HTTP 헤더. 우리는 지금의 PR 이벤트에 관심 있습니다. 거기에서, 우리는거야
정보의 페이로드를 수행하여, 표제 필드를 반환한다. 이상적인 시나리오에서,
우리 서버 풀 요청이 아니라 갱신 될 때마다 관여 될
그것은 열 때. 즉, 모든 새로운 푸시는 CI의 테스트를 통과해야합니다 것입니다.
그러나이 데모를 위해, 우리는 단지 그것을 열 때 걱정됩니다.

테스트에 나뭇 가지에 약간 수정을,이 개념 증명을 테스트하려면
저장소 및 풀 요청을 연다. 서버가 적절하게 응답해야합니다!

## 상태로 작업

대신에 우리의 서버, 우리는 인 우리의 첫 번째 요구 사항을 시작할 준비가
설정 (및 업데이트) CI 상태. 언제든지 당신이 당신의 서버를 업데이트합니다,
당신은 ** 재전송이 ** 같은 페이로드를 전송하실 수 있습니다. 을 할 필요가 없습니다
새 풀 요청하면 변경을 할 때마다!

우리는 API와 상호 작용하고 있기 {{ site.data.variables.product.product_name }} 때문에, 우리는 사용합니다 [Octokit.rb] [octokit.rb]
우리의 상호 작용을 관리 할 수​​ 있습니다. 우리는 함께 그 클라이언트를 구성합니다
[a personal access token][access token]:

```루비
#! EVER 진짜 APP IN 하드 코딩 된 값을 사용하지 마십시오!
# 대신, 다음과 같은 변수를 설정하고 테스트 환경
Access_token은 = ENV ['MY_PERSONAL_TOKEN']

이전에는
@client || = Octokit :: Client.new (: access_token은 => access_token은)
종료
```

그 후, 우리는 명확하게하기 위하여 풀 요청을 업데이트해야합니다 {{ site.data.variables.product.product_name }}
우리는 CI에서 처리하고 있다는 :

```루비
데프 process_pull_request (pull_request)
"처리 풀 요구를 ..."두고
@ ['base'] ['repo'] ['full_name'] client.create_status (pull_request, ['head'] ['sha'] pull_request, '보류')
종료
```

우리는 여기서 세 가지 매우 기본적인 일을하고 있습니다 :

* 우리는 저장소의 이름을 찾고
* 우리는 풀 요청의 마지막 SHA를 찾고
우리가 상태를 설정하는 * "보류"로

즉입니다! 여기에서, 당신은 당신이 실행하기 위해 필요 어떤 프로세스를 실행할 수 있습니다
테스트 스위트 룸입니다. 어쩌면 당신은 젠킨스, 또는 전화로 코드를 전달하는거야
등의 API를 통해 다른 웹 서비스에. 그 후, [Travis] [travis api] 당신은 좋겠
한 번 더 상태를 업데이트해야합니다. 우리의 예에서 우리는 단지` "성공"`로 설정합니다 :

```루비
데프 process_pull_request (pull_request)
@ ['base'] ['repo'] ['full_name'] client.create_status (pull_request, ['head'] ['sha'] pull_request, '보류')
잠이 # 바쁜 일을 ...
@ ['base'] ['repo'] ['full_name'] client.create_status (pull_request, ['head'] ['sha'] pull_request, '성공')
"끌어 오기 요청을 처리!"둔다
종료
```

## 결론

GitHub의에서, 우리는 년 동안 우리의 CI를 관리하는 [Janky] [janky] 버전을 사용했습니다.
기본적인 흐름은 기본적으로 우리가 위에서 구축 한 서버와 동일한 것입니다.
GitHub의에서, 우리는 :

끌어 오기 요청을 만들거나 (는 janky를 통해) 업데이트됩니다 젠킨스에 * 화재
* CI를의 상태에 대한 응답을 기다립니다
코드 * 녹색이면, 우리는 풀 요청 병합

이 통신은 모​​두 우리의 대화방에 다시 깔때기된다. 당신은 할 필요가 없습니다
이 예제를 사용하는 자신의 CI 설정을 구축 할 수 있습니다.
당신은 항상 의지 할 수 [GitHub integrations] [integrations] 있습니다.

[deploy API] / V3 /의 repos / 배포 /
[status API] / V3 /의 repos / 상태 /
[ngrok] : https://ngrok.com/
[using ngrok] / webhooks / 구성 / # 사용-ngrok
[platform samples] : https://github.com/github/platform-samples/tree/master/api/ruby/building-a-ci-server
[Sinatra] : http://www.sinatrarb.com/
[webhook] / webhooks /
[octokit.rb] : https://github.com/octokit/octokit.rb
[access token] : https://help.github.com/articles/creating-an-access-token-for-command-line-use
[travis api] : https://api.travis-ci.org/docs/
[janky] : https://github.com/github/janky
[heaven] : https://github.com/atmos/heaven
[hubot] : https://github.com/github/hubot
[integrations] : https://github.com/integrations
