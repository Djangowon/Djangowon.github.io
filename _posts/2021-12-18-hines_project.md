---
layout: post
current: post
cover:  assets/images/hines.jpeg
navigation: True
title: "몇 십년 후의 집: 하인즈"
date: 2021-12-18 
tags: [Project]
class: post-template
subclass: 'post'
author: mrdw
---
# Hines
`Hines`는 ‘몇 십년 후의 집’이라는 컨셉 하에 우주 속 다른 행성들의 땅을 소개하고 필요한 물품 판매를 목적으로 하는 스토어 사이트입니다. 인원은 프론트엔드 개발자 3명, 백엔드 개발자 3명이며 모든 기능은 직접 구현하였습니다.  
[Hines-Backend-Repo](https://github.com/wecode-bootcamp-korea/27-2nd-Hines-backend)

## 적용 기술
- 어플리케이션 API 서버 : Python 3.8 / Django
- Devops : AWS(EC2, S3, RDS), Linux, Docker
- 프로젝트 관리 : notion, trello

## 담당 기능
- ERD & Modelling 제작
- 메인페이지 & 리스트페이지
    - 카테고리 목록 반환 기능 구현
    - 카테고리 및 서브 카테고리에 따른 프로덕트 리스트를 반환하는 API 구현
    - 검색 기능 구현
- 장바구니 기능(CRUD) 구현
- API Unit test에 적용되는 테스트코드 작성
- AWS(EC2, RDS)를 이용하여 배포

## 코드리뷰
#### products/views.py/CategoriesView 수정 전
```python
class CategoriesView(View):
    def get(self, request):
        offset      = int(request.GET.ge('offset', 0))
        limit       = int(request.GET.ge('limit', 6))
        category_id = request.GET.ge('category_id')
        categories = [{
            'id'      : category.id,
            'name'    : category.name,
            'product' : [{
                'id'                  : productid,
                'name'                : productname,
                'price'               : productprice,
                'brand'               : productbrand,
                'thumbnail_image_url' : productthumbnail_image_url 
                } for  product in Product.objects.filter(sub_category__category_id=categoy_id)]
            } for category in Category.objectsfilter(id=category_id)  [offset:offset+limit]
        ]
        return JsonResponse({'message':'SUCCESS','result':categories},     status=200)
```
  
ProductListView 가 메뉴 안의 카테고리 안의 서브카테고리를 클릭하면 해당 서브카테고리의 프로덕트를 반환하도록 구현했는데, 서브카테고리가 참조하고 있는 카테고리만 클릭해도 카테고리의 전체 프로덕트 리스트 반환이 필요해서 카테고리 뷰를 새로 만들었다. 카테고리 뷰를 따로 만들지 않아도 되는 건지 궁금했는데 코드리뷰를 받고 나서 이미 ProductListView에서 반환하고 있는 상품 정보랑 코드가 굉장히 비슷하기 때문에 CategoriesView는 카테고리 정보만 반환하도록 만들고, ProductListView를 통해서 상품 정보를 받아올 수 있도록 아래와 같이 수정하게 되었다. 
#### 수정 후
```python
class CategoriesView(View):
    def get(self, request):
        categories = Category.objects.all().prefetch_related('subcategory_set')

        results = [{
            'id'   : category.id,
            'name' : category.name,
            'sub_category' : [{
                'id'   : sub_category.id,
                'name' : sub_category.name
            } for sub_category in category.subcategory_set.all()]
        } for category in categories]

        return JsonResponse({'message':'SUCCESS', 'result':results }, status=200)
```
  
카테고리만 전체 반환하는 로직을 짰다가 프론트 주영님과 상의 후 상단에 들어갈 카테고리 외에도 사이드바에 있는 메뉴에서 카테고리와 서브카테고리 전부 필요해서 위처럼 수정했다. 수정을 하면서 바뀌는 코드가 Restful 한 코드인가, 기존의 코드가 Restful하지 않은 코드인가 의문을 가지게 되었는데 결과적으로 수정전 코드도 restful 하지 않은 것은 아니다. restful과 별개로 이미 ProductListView 에서 똑같은 로직으로 프로덕트를 반환하는 기능이 이미 있었기 때문에 CategoriesView 는 굳이 프로덕트를 반환하지 않도록 수정된 것이다.

#### cart/views.py/CartView 수정 전
#### * POST
```python
class CartView(View):
    @login_required
    def post(self, request):
        try:    
            data       = json.loads(request.body)
            product_id = data['product_id']
            quantity   = data['quantity']
            
            cart, created  = Cart.objects.get_or_create(
                user_id    = request.user.id,
                product_id = product_id,
            )
            
            if not created:
                cart.quantity += quantity
                cart.save()
                return JsonResponse({'message' : 'SUCCESS'}, status = 200)
            
            cart.quantity = quantity # 여기
            cart.save()

            return JsonResponse({'message' : 'SUCCESS'}, status = 201)
```
처음 작성했던 CartView에서 cart 데이터가 없어서 새로 생성하는 경우와 이미 cart에 담긴 product가 있어서 새로 post를 할 때 수량만 추가되는 경우로 나눠서 생각했었는데, if문은 필요가 없고 표시해둔 부분만 `cart.quantity += quantity`로 `+`만 추가되면 되는 코드였다. 불필요한 if문을 삭제하고 satus code는 200으로 통일하여 아래와 같이 수정했다.
```python
class CartView(View):
    @login_required
    def post(self, request):

        data       = json.loads(request.body)
        product_id = data['product_id']
        quantity   = data['quantity']

        cart, created  = Cart.objects.get_or_create(
            user_id    = request.user.id,
            product_id = product_id
        )

        cart.quantity += quantity
        cart.save()

        return JsonResponse({'message':'SUCCESS'}, status=200)
```
#### * DELETE
```python
def delete(self, request):
    try: 
        data = json.loads(request.body)
        Cart.objects.filter(id__in=data['cart_id']).delete()
        return JsonResponse({'message':'SUCCESS'}, status=200)
    
    except KeyError:
        return JsonResponse({'message : KEY_ERROR'}, status = 400)
```
  
장바구니의 삭제 기능이다. 처음에 작성할 땐 id__in 이 'id 를 전체를 불러오고 같은 cart_id 가 있으면 삭제' 이렇게 생각해서 위처럼 코드를 짰는데 이상한 코드가 됐다. 만약 cart_id를 받아서 cart 한 개를 삭제하는 로직은 path parameter를 사용해야 하고, cart_id 를 여러개 받아서 해당 id 값의 cart를 삭제하려면 query parameter를 사용해야 하는 것을 알게 되었다.  
*하나의 parameter -> path, 여러개의 parameter -> query

#### 수정 후 
```python
def delete(self, request):
    cart_ids = request.GET.get('cartIds')

    Cart.objects.filter(id__in=cart_ids, user_id=request.user).delete()

    return JsonResponse({'message':'NO_CONTENTS'}, status=204)
```
```python
    # cart 한 개 지우는 로직 -> path parameter 사용
    def delete(self, request, cart_id):
        Cart.objects.filter(id=cart_id, user_id=request.user).delete()
        return JsonResponse({'message':'NO_CONENTS'}, status=204)
    
    # DELETE :8000/carts?cartIds=[1,2,3,4]  
    def delete(self, request):
        cart_ids = request.GET.get("cartIds")
        Cart.objects.filter(id__in=cart_ids, user_id=request.user).delete()
        return JsonResponse({'message':'NO_CONENTS'}, status=204)
```
#### + 
#### path parameter 적용완료한 patch 로직
```python
def patch(self, request, cart_id):
        try:
            data = json.loads(request.body)

            cart = Cart.objects.get(id=cart_id, user=request.user)

            cart.quantity = data['quantity']
            cart.save()

            return JsonResponse({'message':'SUCCESS'}, status=200)

        except KeyError:
            return JsonResponse({'message':'KEY_ERROR'}, status=400)
```

## TeamHines 회고
2차 프로젝트가 `오늘의 집` 서비스로 결정되고 처음에 팀원들과 사이트를 둘러보며 우리 프로젝트명과 어떤 기능을 구현할지 스프린트 미팅을 했다. 여러 의견들 중에서 스쳐지나가듯 `몇십년 후의 집`이 나왔는데 처음엔 웃기기도 하고 단번에 그 문장에 꽂혀서 `House in decades`->`Hines(하인즈)`로 결정되었고 우주 행성 커머스 사이트로 기획하게 되었다. 우리가 구현할 기능 중에 오늘의 집 사이트에서 크게 스토어와 커뮤니티로 나누어졌는데 프로젝트 기간에 맞추어 스토어만 구현하는 것으로 목표를 설정했다. 웃으며 한 말인데 기획이 산으로 가는 것도 아니고 우주로 갔다고 했지만 미팅 분위기는 좋았고 모두 우주 사이트를 멋있게 만들어보자고 시작했던 기억이 난다. 명확한 컨셉으로 인해 결과적으로 우리 사이트에서 오늘의 집의 흔적은 찾을 수 없게 되었다. (그냥 다른 커머스 사이트 같음!)

자유로운 분위기로 회의록도 서기를 따로 정하지 않고 본인이 맡은 부분을 각자 적어서 미팅때 다같이 보며 노션페이지와 트렐로를 보며 이야기했고, 미팅시간도 1주차까지는 유동적으로 조율하면서 진행했는데 시간은 고정시간을 정해놓고 하는 게 좋은 것 같아서 2주차부터 고정 시간에 일일 스크럼을 하는 걸로 바꾸었다. 

이번 프로젝트때 최종발표 이틀 전까지 프.백 모두 기능구현이 완료되지 않아서 통신이 많이 늦어졌고 맞춰보는데 시간소요가 예상보다 많이 되었었다. 하루 전 날 프론트와 맞춰보면서 에러를 고치며 수정한 부분도 많았기 때문에 그동안 프론트와 제대로 소통한 게 아니었구나 생각했다. 프로젝트를 마무리하고 회고 미팅을 할 때 한 프론트 팀원분의 말이 '아쉬운 점은 백엔드와 소통을 많이 안 한 것, 아이러니하지만 잘한 점도 백엔드와 소통을 원활하게 한 것' 인데 그게 특정한 같은 기능을 맡은 프.백 멤버들끼리만(특정 팀원이랑만) 맞춰보고 이야기해서 다른 팀원이랑 또 다른 기능을 통신할 때 걸림돌이 많았던 게 나 역시 그랬던 것 같아서 아쉬운 점으로 꼽았다. 

