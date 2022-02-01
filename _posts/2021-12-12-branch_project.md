---
layout: post
current: post
cover:  assets/images/branch.jpg
navigation: True
title: "첫 프로젝트: 코드가 작품이 되는 공간, 브랜치"
date: 2021-12-12 
tags: [Project]
class: post-template
subclass: 'post'
author: mrdw
---
# Branch
카카오에서 서비스 중인 작가들의 블로그 사이트 Brunch를 모티브로 하여 프로젝트를 진행하였습니다.
프론트엔드 개발자 3명, 백엔드 개발자 3명으로 이루어진 `Branch` 프로젝트는 개발자들의 기술 블로그 컨셉으로 기존의 사이트를 참고하여 모든 기능은 직접 구현하였습니다.  
[Branch-Backend-Repo](https://github.com/wecode-bootcamp-korea/27-1st-branch-backend/)

## 적용 기술
- 어플리케이션 API 서버 : Python 3.8 / Django
- Devops : AWS(EC2, S3, RDS), Linux
- 프로젝트 관리 : notion, trello

## 담당 기능
- ERD & Modelling 제작
- JWT 라이브러리를 이용한 사용자 인증 API 작성
- login_required, public 데코레이터 모듈화
- 키워드와 태그에 따른 포스트 리스트 페이지 목록 구현
- 좋아요 토글 기능 구현
- Faker 라이브러리를 이용한 MySQL 더미데이터 삽입
- AWS(EC2, RDS)를 이용하여 배포

### 코드리뷰
```python
class LikeView(View):
    @login_decorator
    def post(self, request):
        try:
            data = json.loads(request.body)
            user = request.user

            like, created = Like.objects.get_or_create(user_id = user.id, posting_id = data['posting_id'])

            if not created:

                like.delete()
                return JsonResponse({'message':'CANCELED_LIKE'}, status=201)

            return JsonResponse({'message':'SUCCESS_LIKE'}, status=200)

        except JSONDecodeError:
          return JsonResponse({'message':'JSON_DECODE_EEROR'}, status=400)
```
  
`get_or_create` 메서드를 사용해본 좋아요 기능이다. 이 메서드는 (object, created) 라는 튜플 형식으로 반환하고, 첫 번째 인자(object)는 꺼내려고 하는 모델의 인스턴스이다. 두 번째 인자(created)는 boolean flag로, TRUE 또는 FALSE를 갖고 있다. `get_or_create` 메서드에 의해 인스턴스가 생성되었다면 TRUE, 데이터베이스에 존재하던 인스턴스라면 FALSE 값을 가진다.

#### PostListView 수정 전
```python
class PostListView(View):
    def get(self, request, keyword_id):
        order_method = request.GET.get('sort_method', 'created_at')
        limit        = int(request.GET.get('limit', 100))
        offset       = int(request.GET.get('offset', 0))

        posts = Posting.objects.filter(keyword_id=keyword_id).select_related('user').order_by(order_method)[offset:limit]

        results = [{
            'title'      : post.title,
            'sub_title'  : post.sub_title,
            'content'    : post.content,
            'thumbnail'  : post.thumbnail,
            'user'       : post.user.nickname,
            'created_at' : post.created_at, 
            'tag'        : list(post.keyword.postingtag_set.values('name')) } for post in posts
            ]

        return JsonResponse({'result':results}, status=200)
``` 
#### PostListView 수정 후
```python
class PostListView(View):
    def get(self, request, **kwargs):
        try :  
            order_method = request.GET.get('sort_method', 'created_at')
            limit        = int(request.GET.get('limit', 100))
            offset       = int(request.GET.get('offset', 0))

            q= Q()

            if kwargs :
                q  &=Q(keyword_id=kwargs['keyword_id'])

            posts = Posting.objects.filter(q).select_related('user').order_by(order_method)[offset:limit]

            results = [{
                'id'        : post.id,
                'title'     : post.title,
                'sub_title' : post.sub_title,
                'content'   : post.content,
                'thumbnail' : post.thumbnail,
                'user'      : post.user.nickname,
                'created_at': post.created_at,
                'tag'       : list(post.keyword.postingtag_set.values('name')) } for post in posts
                ]

            return JsonResponse({'result':results}, status=200)
        except KeyError :
            return JsonResponse({'MESSGE':'KEY_ERROR'}, status=400)
```
  
기존에 키워드 id에 따른 포스트 리스트뷰를 작성해놓았는데, 메인페이지에는 parameter가 필요없기 때문에 다른 url을 만들어서 기능 구현하는 것이 필요했다. 이런 경우 조건을 다르게 줄 수 있는 Q객체를 사용하는 것이 적합하다고 리뷰를 받아 수정된 코드이다. 

#### 재밌었던 Faker 라이브러리, db_uploader.py
  
```python
import os
import django
import csv
import random

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "teambranch.settings")
django.setup()

from users.models       import User
from keywords.models    import Keyword
from branch_tags.models import UserTag, UsersUserTags, PostingTag, PostingsPostingTags
from postings.models    import Posting
from faker              import Faker

fake = Faker()

for _ in range(100000):

    email         = fake.unique.email()
    profile_photo = 'https://raw.githubusercontent.com/Djangowon/TIL/main/image/8dd4f6a781571f85eac240b0f31ecfa3.jpeg'
    position_list = ['Backend', 'Frontend', 'Fullstack']

    if not User.objects.filter(email=email).exists():
    
        User.objects.create(
            name          = fake.name(),
            nickname      = fake.first_name(),
            email         = email,
            password      = fake.password(),
            phone_number  = fake.unique.phone_number(),
            github        = fake.email(),
            profile_photo = profile_photo,
            description   = fake.text(),
            position      = random.choice(position_list),
            created_at    = fake.date_time(),
            updated_at    = fake.date_time()
        )

    else:
        print(email)

    usertag_name = fake.first_name()

    if not UserTag.objects.filter(name=usertag_name).exists():
        
        UserTag.objects.create(
            name       = usertag_name,
            created_at = fake.date_time(),
            updated_at = fake.date_time()
        )
    
    else:
        print(usertag_name)
    
for _ in range(20000):
    UsersUserTags.objects.create(
        user_id     = fake.pyint(min_value=7,max_value=100000),
        user_tag_id = fake.pyint(min_value=1,max_value=720)
    )

for _ in range(100000):

    thumbnail = 'https://raw.githubusercontent.com/Djangowon/TIL/main/image/15C58535-76A3-4A64-813A-3896D4A6DEE7.jpeg'
    
    Posting.objects.create(
        title        = fake.name(),
        sub_title    = fake.name(),
        content      = fake.text(),
        thumbnail    = thumbnail,
        created_at   = fake.date_time(),
        updated_at   = fake.date_time(),
        keyword_id   = fake.pyint(min_value=1,max_value=19),
        user_id      = fake.pyint(min_value=7,max_value=100000)
    )

    postingtag_name = fake.first_name()

    if not PostingTag.objects.filter(name=postingtag_name).exists():

        PostingTag.objects.create(
            name         = postingtag_name,
            created_at   = fake.date_time(),
            updated_at   = fake.date_time(),
            keyword_id   = fake.pyint(min_value=1,max_value=19)
        )

    else:
        print(postingtag_name)

for _ in range(20000):
    PostingsPostingTags.objects.create(
        posting_id     = fake.pyint(min_value=50,max_value=100000),
        posting_tag_id = fake.pyint(min_value=1,max_value=720)
    )
```
  
최대한 모델링을 수정하지 않도록 고민해서 했지만 중간에 수정이 조금씩 생겼다. 아직 서툴러서 중복되는 데이터도 생기고 models.py수정도 몇 번 있었어서 migrations 파일과 데이터베이스를 여러번 밀고 다시 빌드하고를 반복하면서 원없이 데이터베이스를 가지고 놀 수 있었다. 테이블당 10만개의 데이터를 넣어보자 해서 Faker 라이브러리를 사용해봤는데 SQL문을 더 파보고 싶다는 생각이 들 정도로 재미있는 작업이었다.

## TeamBranch 회고
한달동안 기초 개념들을 공부하면서 나는 아직 모르는 게 많은데 프로젝트를 할 수 있을까에 대한 고민도 많았고 이게 맞나 생각이 끊임없이 들어서 걱정이 많은 상태로 1차 프로젝트를 진행하게 되었다. 개발을 처음 해보고 언어와 프레임워크 모두 낯설어서 시간을 더 갈아넣기로 했다. (이때까지만 해도 프로젝트 최종발표 전 날 가장 늦게까지 남아있던 팀이 우리팀 일 줄은 몰랐다ㅎ)  
  
프로젝트 진행상황을 공유하고 소통하는 방법으로 트렐로와 노션을 활용했고 일일 스크럼, 일일 스탠드업 미팅을 진행했다. 어제 한 일, 오늘 할 일, 블록커를 공유했고 이 미팅으로 정말 우리가 개발을 하고 있구나를 실감했다. 백엔드의 경우 모델링 수정을 반복하면서 목요일부터 제대로 기능 구현을 시작한 반면에 프론트 세 분이 매일 미팅에서 보여주는 화면이 뚝딱뚝딱 만들어지는 게 보여서(흔들리지 않는 편안함..) 너무 든든했다. 이 페이지가 우리가 만드는 웹이구나 하고.  
  
로직을 구현하는 데 있어서 무엇이 맞는 것인지 몰라 확신이 없었고 중간 중간 바로 이해가 되지 않는 부분도 있어 막히고 좌절하는 순간도 있었다. 한번은 이해가 잘 되지 않아서 답답할 때가 있었는데 옆에서 응원과 힘이 되는 조언을 해주는 동기분들이 있어서 정말 든든했고, 나도 팀원들에게 그런 영향력을 줄 수 있는 사람이 되고 싶다고 생각했다. 각자 맡은 기능이 다르기 때문에 내가 이 기능을 구현할 때 코드는 어떻게 썼고 왜 이렇게 짰는지 설명하고 공유하면서 배우는 게 많았고 더 개발문화에 빠져드는 것 같다. 어떤 개발자든 똑같은 마음이겠지만 나도 역시 '함께 일하고 싶은 개발자'가 되고 싶다. 첫번째 프로젝트에서 정말 많은 것을 얻었는데 가장 큰 수확은 멘탈과 마인드셋을 잘 다지게 되었다.

프로젝트 중간에도 팀원들과 이야기하면서 우리가 놓치고 있는 부분, 개선할 부분에 대해서도 많은 이야기를 나누었다. 백엔드 공통으로 아쉬웠던 점은 처음에 역할분담을 할 때 필수 구현 사항들이 막상 브런치 페이지가 담백한 기능들이 많았기 때문에 프론트 처럼 페이지 기준으로 나누는 것과 백엔드 테이블이나 기능 기준으로 나누는 것에 대해 고민이 많았다. 그 점에서 처음에 명확하게 역할분담이 되지는 않았어서 좀 아쉬웠다. (맡았던 기능 구현을 끝내면 추가구현으로 무엇을 할 지 다시 분배하고 그 전까지 붕 뜨는 느낌)
또, 처음 기능구현을 하면서 기능별로 마감 기한을 정해놓지 않았어서 압박감이 없었던 것은 장점이지만 계획적이지 않았던 것 같아서 아쉬웠다. 그래서 계속해서 그때그때 소통하면서 해결했지만 바로 이야기하면서 해소한 부분들이 많아서 기록을 많이 하지 못 한 것이 아쉽다.

좋았던 점은 무엇보다 팀의 분위기가 좋았다. 프.백끼리 데이터를 어떻게 주고받을 것인지 소통할 때나 어떤 기능을 어떤 방식으로 맞춰볼 것인지 조율할 때에도 서로가 필요한 것을 채워주고 해결해주었다. '이게 필요한데 이 부분 어려울까요?' 하면 '필요해요? 만들어 드릴게요!' 거의 이런 식.  
문제 해결을 함에 있어서 함께 고민하고 계속 소통하면서 공유했는데 우리 팀이 '이건 또 뭐냐'하면서 에러가 나도 웃고 있으니까 다른 동기분이 싱글벙글팀 이라고ㅋㅋㅋ 누가 우리 팀의 장점이 뭐냐고 물어봤을 때 '싱글벙글팀' 이라고 바로 말하게 되었다. 둥글둥글한 팀브랜치 동기분들과 함께 해서 진짜 재밌게 개발했고 감사합니다. 꼭 다시 모여서 리팩토링하시죠. 브랜치 ㅎㅇㅌ(이 브랜치 화이팅을 2차 프로젝트 진행 중인 지금까지도 외치고 있다ㅎ)  

### 데모
<!-- <img src="{{ '/assets/img/' }}" alt=""> -->
gif로 변환했더니 속도가 상당히 느리다. 시간 날 때 부분 gif 로 올려야겠다ㅎ

