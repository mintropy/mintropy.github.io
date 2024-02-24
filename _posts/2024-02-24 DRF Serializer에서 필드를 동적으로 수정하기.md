---
title: DRF Serializer에서 필드를 동적으로 수정하기
categories:
  - python
  - django
tags:
  - python
  - django
date: 2024-02-24T15:20:00+09:00
---
# DRF 하나의 Model을 다양한 배경에서 사용

```python
class BaseModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

class Community(BaseModel):
    title = models.CharField(max_length=100)

class Post(BaseModel):
    community = models.ForeignKey(
        Community, on_delete=models.SET_NULL, null=True, related_name="posts"
    )
    title = models.CharField(max_length=100)
    content = models.TextField()
```

위와 같이 모델을 구성했다고 하자. 두 개의 모델, Community, Post에 대하여 각각 생성, 수정 날짜가 기록되고, Community는 이름만을 추가로 가지고, Post는 외래키로 Community와 연결되어 제목과 내용을 각각 가진다. 게시판을 동적으로 생성할 수 있고, 게시판별로 글이 작성되는 구조로 생각할 수 있다. (이하, 커뮤니티, 게시글 이라고 부르겠다.)

```python
class CommunitySerializer(serializers.ModelSerializer):
    post_count = serializers.IntegerField(read_only=True)
    latest_post = serializers.DateTimeField(read_only=True)
    new_community = serializers.BooleanField(read_only=True)

    class Meta:
        model = Community
        fields = "__all__"

class CommunityAPI(APIView, PageNumberPagination):
    def get(self, request: Request) -> Response:
        seven_days_ago = datetime.now() - timedelta(days=7)

        communities = (
            Community.objects.all()
            .annotate(
                post_count=Count("posts"),
                latest_post=Max("posts__created_at"),
                new_community=Case(
                    When(created_at__gte=seven_days_ago, then=True),
                    default=False,
                    output_field=BooleanField(),
                ),
            )
            .order_by("-created_at")
        )

        paginated_communities = self.paginate_queryset(communities, request, view=self)
        serializer = CommunitySerializer(paginated_communities, many=True)
        return self.get_paginated_response(serializer.data)
```

커뮤니티 목록을 받는 API를 위와 같이 생성한다고 하자. 각 커뮤니티별로 추가적으로 필요할 수 있는 정보를 같이 담아서 보낸다. 예를 들어 커뮤니티의 게시글 갯수, 최신 게시글 날짜, 최근 생성된 커뮤니티인지 등을 annotate로 생성하여 반환한다. 그리고 반환은 아래와 같다.

```json
{
  "count": 5,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": "18741e30-da8f-4cc9-88e3-20dd0f039b20",
      "post_count": 0,
      "latest_post": null,
      "new_community": true,
      "created_at": "2024-02-20T12:20:49.690184Z",
      "updated_at": "2024-02-20T12:20:49.690212Z",
      "title": "5"
    },
    ...
  ]
}
```

지금은 예시를 위해 간단한 모델을 생성하여 정보가 많지 않지만, 실제 서비스를 하게 되고, 서비스 기간이 늘어나게 되면 더 다양한 정보들을 가지게 되고, 반환하게 된다. 그런데 어떤 때는 저러한 정보들 중 일부만 필요하다거나, 어떤 정보는 일반 사용자에게 공개되지 않는 것이 좋을 수 있다. 여기서는 new_community 필드는 어드민 페이지에서는 필요없고, post_count, created_at, updated_at은 유저 페이지에는 보여주고 싶지 않은 정보라고 가정해보자. 이런 조건을 만족시키기 위해서는 여러개의 serializer를 만드는 방법이 있다. 각 상황별로 필요한 필드만을 정의한 serializser를 만드는 방법이다. 하지만 이 경우 코드 중복이 많을 수 있어 상속을 활용하는 방법도 있다. 하지만 이번에는 조금 더 간단할 수 있는 방법으로 접근해보자

# Serializer 필드를 동적으로 지정하기

```python
class CommunitySerializer(serializers.ModelSerializer):
    post_count = serializers.IntegerField(read_only=True)
    latest_post = serializers.DateTimeField(read_only=True)
    new_community = serializers.BooleanField(read_only=True)

    class Meta:
        model = Community
        fields = "__all__"

    def __init__(self, *args, **kwargs) -> None:
        context = kwargs.get("context", None)
        super().__init__(*args, **kwargs)
        exclude_fields = []
        if context == "admin":
            exclude_fields = ["new_community"]
        elif context == "user":
            exclude_fields = ["post_count", "created_at", "updated_at"]
        for field_name in exclude_fields:
            self.fields.pop(field_name)
```

간단하게 요약하면, 다음과 같이 `__ini__`메서드를 오버라이드하여 결과를 달성할 수 있다. context라는 변수를 입력하여 admin, user에 따라 특정 필드를 제거하는 것이다.

```python
{
   'id': UUIDField(read_only=True),
   'post_count': IntegerField(read_only=True),
   'latest_post': DateTimeField(read_only=True),
   'new_community': BooleanField(read_only=True),
   'created_at': DateTimeField(read_only=True),
   'updated_at': DateTimeField(read_only=True),
   'title': CharField(max_length=100)
 }
```

위에서 `self.fields.pop()`형식으로 지정하는데, 그것이 가능한 이유는 serializer에서 필드는 위와같이 딕셔너리 형태로 구성되어있기 때문이다. 그래서 필요에 따라 특정 필드를 제거하거나 추가할 수 있다. 제거할때는 위와같이 딕셔너리 pop 메서드를 사용하면 되고, 추가할때는 `fields["key"]=Field`형식으로 추가하면 된다. 여기서 Field는 serializers.Feild 클래스가 된다.
Django는 필요할 때 SQL query를 실행하게 되어 serializer 필드를 조정하는 순간까지도 실행하지 않다가, 직렬화를 위해 데이터가 필요한 순간 데이터를 가져오게 된다. 즉, 필요에 따라 특정 필드들을 제외하는것이 성능향상에도 도움될 수 있다. 때로는 ForeignKey, ManyToMany 등으로 하나 또는 둘 이상 연결된 테이블을 정보를 쓰는 경우가 있는데, 그러한 정보가 필요하지 않은 경우도 있을 수 있다. 커뮤니티 목록을 단순하게 네비게이션 바나 사이드 바 정도에서만 보여주는 경우 이름만 필요한데, 이 때 게시글과 관련된 필드를 모두 제외하면 join 이나 prefetch 등이 애초에 필요없어지게 된다.