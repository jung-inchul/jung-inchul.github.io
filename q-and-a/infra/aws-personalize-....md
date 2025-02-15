# AWS Personalize 적용 후기… 😰

## AWS Personalize 란?

* AWS 자체 구현된 머신러닝 기술을 이용하여 사용자가 데이터만 주입하면 개인화 추천 시스템을 제공해주는 서비스이다
* AWS Personalize 의 가장 매력적인 부분은 사용자가 머신러닝 구현 기술에 대한 이해가 없어도 몇 가지 설정만으로 머신러닝 기술을 사용할 수 있다는 점이다
* 그리고 단순히 추천을 해주는게 아닌 서비스 특성에 맞는 추천 시스템을 제공할 수 있다는 점이 장점이라고 할수 있다 (영상/커머스/사용자 커스텀)

<figure><img src="../../.gitbook/assets/1 (14).png" alt=""><figcaption></figcaption></figure>

## AWS 사용사례

* AWS Personalize 가이드 페이지를 살펴보면 실제로 많은 고객사들이 사용한 사례를 살펴볼 수 있다
  * 배달의 민족은 AWS Personalize를 사용하여 기존 클릭수를 24% 증가시키고 추천을 통한 작품 만족도도 15% 증가하는 결과를 얻었다고 한다
  * Brandi는 AWS Personalize를 도입한 이후에 클릭수가 8.6K → 9.8K까지 증가하였고, 방문자마다 상품 클릭률도 15% 정도 증가되었다는 성과를 공유하였다
  * 롯데마트는 AWS Personalize 도입 이후에 쿠폰 사용량이 2개 이상 증가하고 신제품 구매 빈도가 1.7배 증가하였다고 한다

{% hint style="info" %}
AWS Personalize 사용사례

[https://aws.amazon.com/ko/blogs/korea/category/artificial-intelligence/amazon-personalize/](https://aws.amazon.com/ko/blogs/korea/category/artificial-intelligence/amazon-personalize/)
{% endhint %}

## 우리가 제공하고 싶은 기능은 무엇인가?

* 우리는 사용자의 인터랙션에 따라서 어떤 취향의 작품을 좋아하는지 결과를 도출하고 관련 컨텐츠를 제공하는 기능이 필요하다.
* 사용자에게 좀더 다양한 컨텐츠를 제공함으로써, 신규 작가님들에게 사용자 유입을 증가시키고, 사용자의 이탈률을 줄이고 활성 유지 시간을 증가시키는 것이 개인화 추천 기능의 목적이다.

## 그럼 AWS Personalize를 어떻게 구성하면 될까?

1. 우선 데이터 그룹부터 생성해야 한다. 유형은 커스텀으로 생성한다
2. 초기 기계학습에 필요한 데이터를 주입한다.
   1. S3에 CSV 형태로 저장해서 Personalize dataset 경로로 지정해주면 된다. ( item, user는 부가정보라 기계학습에 필요한 필수 데이터는 user\_intercation 데이터이니 해당 데이터만 생성해도 우선은 솔루션을 생성할 수 있다 )
3. 솔루션을 생성한다
   1. 솔루션 유형은 아이템 추천을 선택한다
   2. 레시피는 aws-user-personalize 를 선택한다.
      1. aws-sims : 인터랙션 기반으로 가장 유사한 아이템 순위 노출
      2. aws-similar-items : 인터랙션 기반으로 가장 유사한 아이템의 같은 카테고리의 아이템 순위 노출
      3. aws-personalized-ranking : 선별된 목록내에서 아이템 순위를 노출
      4. aws-user-personalization : 사용자 인터랙션 기반으로 사용자별 아이템 순위를 노출
      5. aws-trending-now : 최근 특정 시간 범위내에 가장 많은 인터랙션이 발생한 아이템 순위를 노출
      6. aws-popularity-count : 가장 많은 인터랙션이 발생한 아이템 순위 노출
   3. 옵션은 기본 옵션을 사용한다 (aws-user-personalize 레시피에 해당 하는 옵션)
      1. HPO : false ( 머신러닝할때 최적화할 옵션들을 설정할 수 있다. 처음엔 굳이 사용하지 않아도 된다 )
      2. User history length percentile : { min : 0.00, max : 0.99} ( 사용자의 전체 인터랙션 기록에서 몇 %의 범위를 사용할지 정의한다)
      3. Backpropagation through time : 32 ( 과거 히스토리를 얼마나 오랫동안 반영할지를 결정하는 수치이다. 수치가 작을 수록 최근 데이터를 더 많이 고려하여 계산하고, 수치가 클 수록 과거 데이터를 참고하여 계산하게 된다. )
      4. Hidden dimension : 149 ( 사용자와 아이템 간의 시퀀스 정보를 고려하여 수치가 클수록 복잡성이 커져 계산이 오래 걸리며 수치가 작을수록 계산이 빠르다)
      5. explorationWeight : 1 ( 이전의 인터랙션이 아닌 새로운 상품 위주로 추천해주고 싶을 경우, 0값이면 새로운 값은 추천하지 않는다 )
4. 캠페인을 생성한다
   1. TPS(Transaction Per Seconds) : 100 (TPS는 초당 처리할 수 있는 요청수를 의미한다. 처리수가 낮을수록 계산이 오래걸리게 되며 요청에 대한 응답 대기 시간이 발생할 수 있다. 최댓값이 500인데 무조건 많이 처리하는건 비용적인 측면에서 불필요한 지출일 수 있으니 서비스 특성에 따라 적절한 값을 정하는게 중요하다)
5. 이벤트 트래커를 생성한다
   1. 이벤트 트래커를 사용하는 이유는 실시간으로 사용자 인터랙션을 반영하여 아이템 순위를 갱신하기 위함이다.

## AWS Personalize를 어떻게 구성하면 좋을까?

<figure><img src="../../.gitbook/assets/2 (11).png" alt=""><figcaption></figcaption></figure>

### 집계 데이터 수집하기

1. 기존에 제공하던 API에서 반영하고 싶은 인터랙션을 취합하여 메시지큐로 전달한다
2. 메시지큐에 전달된 데이터를 AuroraDB에 저장하여 기록을 남겨둔다
3. 이벤트 트래커에 인터랙션 데이터를 호출하여 실시간으로 집계에 반영하도록 한다

{% hint style="info" %}
**사용자 집계를 하기 전에 주의할 점!!**

\
실시간 데이터만으로는 ML에 필요한 데이터가 부족하여 집계를 할수 없다. 초기에는 인터랙션 데이터를 CSV로 작성하여 최소 1000개의 데이터가 필요하며 아이템과 사용자도 최소 30개 이상으로 작성해야 집계를 할수 있는 상태가 된다.
{% endhint %}

### 집계 데이터 사용하기

1. 사용자 개인화 추천 기능을 호출하면 AWS Personalize에게 userId 기반으로 취향을 조회한다
2. AWS Personalize에서 전달된 취향 기반으로 API에서 데이터를 가공하여 사용자에게 노출한다

## 사용하면서 경험한 몇 가지 주의사항

### AWS Personalize는 만능이 아니다

* 실제로 사용해보니 도출해주는 집계 데이터가 100% 정확하지는 않았다
* event value에 더 높은 값을 부여해보거나 비중을 추가하고 싶은 만큼 빈도수를 늘려보았지만 의도한 대로 집계를 내려주지는 않았다…
* 처음엔 집계할 데이터 군집이 적기 때문에 정확하지 않을 것이라 생각하고 우선 적용을 하고 시간을 두고 기다려보기로 하였다..
* 그리고 집계에는 존재하지 않는 신규 데이터가 순위에 집계된 경우도 있었다. 이는 explorationWeight 값을 0으로 설정하여 해결할 수 있었다.

#### 집계 데이터

| user id | item id | event value |
| ------- | ------- | ----------- |
| 1       | 1       | 10          |
| 1       | 2       | 20          |
| 1       | 2       | 20          |
| 1       | 2       | 20          |
| 1       | 1       | 20          |
| 1       | 3       | 10          |

#### 결과값

```jsx
[
	{
		"itemId" : 3,
		"score" : 0.62***
	},
	{
		"itemId" : 2,
		"score" : 0.23***
	},
	{
		"itemId" : 4, // 이 아이템ㅇㄴ 넣어준적도 없는데..? 1번보다 더 높다..?ㅜㅜ
		"score" : 0.10***
	},
	{
		"itemId" : 1,
		"score" : 0.05***
	}
]
```

### 비용적인 측면도 고려해야 한다

* 트래픽에 따라서 실시간으로 기계학습을 수행하기 때문에 인터랙션이 추가될 때마다 비용이 추가된다.
* 비용 계산기로 대략적인 금액을 계산해보자
  * 월별 평균 수집 데이터 : 200 GB
  * 월별 평균 훈련 시간 : 10만건 (TPS당 훈련하는 횟수)
  * 데이터 세트의 사용자 수 : 200명
  * 월 비용 : 2,410 UDS(대략 250만원)
* 초기에 사용자 유입이 많지는 않은데 Personalize를 도입해서 기대하는 효과를 볼수 있는 환경이 아니라면 도입 시기를 늦추는게 좋을 듯 하다 (체감상 비용 부담이 많이 된다..ㅜㅜ)



<figure><img src="../../.gitbook/assets/3 (9).png" alt=""><figcaption></figcaption></figure>

## AWS Personalize를 사용해보며 내린 결론

* AWS Personalize를 처음에 적극 도입하려던 이유는 10분 이내에 개인화 추천 시스템을 구축할 수 있다는 장점이었다
* 실제로도 몇번의 클릭으로 바로 구축할수는 있었다.
* 그러나 주의사항에서도 작성하였는데 우리가 의도한 대로 랭킹이 집계되지 않았고, 기계학습에 맞게 데이터나 설정 정보를 변경하더라도 내부는 블랙박스로 되어있어 커스텀하는데에는 한계가 있었다…ㅜ
* 그리고 개인화 추천기능으로 유입된 사용자에 대한 마케팅 전략이지 신규 사용자를 유입하기란 힘든 부분이 있어서 사용자가 적은 앱에서 Personalize를 제공하기에는 비용적인 부담이 사실 많이 컸다
* 결론적으로는 Personalize를 걷어내고 우리가 의도한 알고리즘대로 수행할 수 있게 직접 구현하기로 하였다.
* 이미 Event tracker에 전송하기 위해 사용자별 취향 점수를 계산하고 있었기 때문에 집계하는 로직만 추가적으로 구현하면 되기 때문에 그렇게 큰 작업은 아니었다.
* 확실히 thrid party나 라이브러리를 사용하는건 편리하지만 그 만큼 우리 입맞에 맞추는건 더 많은 노력과 시간이 필요하고 어느 경우엔 라이브러리에 맞춰서 기획 내용을 변경해야 하는 경우도 생기는건 어쩔수 없는것 같다는 생각이다.

## 참고

* [https://aws.amazon.com/ko/personalize/](https://aws.amazon.com/ko/personalize/)
* [https://www.awsstartup.io/ai-ml/personalized-recommendations](https://www.awsstartup.io/ai-ml/personalized-recommendations)
* [https://www.megazone.com/awstechblog\_221226/](https://www.megazone.com/awstechblog\_221226/)
* [https://labs.brandi.co.kr//2019/10/04/dev1team.html](https://labs.brandi.co.kr/2019/10/04/dev1team.html)
