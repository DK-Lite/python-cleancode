## Python의 데코레이터
기존 방식의 문제점은 함수의 변형을 할 때마다 modifier 함수를 사용하여  
함수를 호출한 뒤 다음 함수를 처음 정의한것과 동일하여 재할당 해야한다.

```python
original = modifier(original)
```

위 방식은 아래의 이류로 혼란스럽고 오류가 발생하기 쉽고 번거롭다.
- 함수 재할당을 잊어버림
- 함수 정의가 멀리 떨어질 경우

**데코레이터는 아래와 같이 변경가능**
```python
@modifier
def original(...):
    ...
```
- 함수, 메서드, 제너레이터, 클래스에 적용가능

> 데코레이터라는 이름은 래핑된 함수의 기능을 수정하고 확장하기 때문에  
정확한 이름이지만 "데코레이터 디자인 패턴"과 혼동하면 안된다.  


### 함수 데코레이터
도메인의 특정 에외에 대해서 특정 횟수만큼 재시도함는 예제
```python
class ControlledException(Exception):
    """도메인에서 발생하는 일반적인 예외"""

def retry(operation):
    @wraps(operation)
    def wrapped(*args, **kwargs):
        last_raised = None
        RETRIES_LIMIT = 3
        for _ in range(RETRIES_LIMIT):
            try:
                return operation(*args, **kwargs)
            except ControlledExeption as e:
                logger.info(f"retrying {operation.__qualname__}")
        raise last_raised
    return wrapped

@retry
def run_operation(task):
    """실행 중 예외가 발생할 것으로 예상되는 특정 작업을 수행"""
    return task.run()
```

### 클래스 데코레이터
- 데코레이터 함수의 파라미터로 함수가 아닌 클래스를 받음
- 클래스 데코레이터는 코드 재사용과 DRY원칙의 모든 이점을 공유
- 여러 클래스가 특정 인터페이스나 기준을 따르도록 강제할수 있음
- 여러 클래스에 적용할 검사를 데코레이터에서 한번만 하면됨

로그인 플랫폼에서 클래스 직렬화 예시
```python
class LoginEventSerializer:
    def __init__(self, event):
        self.event = event
    
    def serialize(self) -> dict:
        return {
            "username": self.event.username,
            "passord": "민감한 정보",
            "ip": self.event.ip,
            "timestamp": self.event.timestamp.strftime("%y-%m-%d %H:%M")
        }

class LoginEvent:
    SERIALIZER = LoginEventSerializer

    def __init__(self, username, password, ip, timestamp):
        self.username = username
        self.password = password
        self.ip = ip
        self.timestamp = timestamp
    
    def serialize(self) -> dict:
        return self.SERIALIZER(self).serialize()
```

시스템을 확장할수록 아래와 같이 문제점이 발생
- 클래스가 많아짐 (이벤트클래스와 직렬화 클래스가 1:1매핑이 되어있어 직렬화 클래스가 많아진다.)
- 유연하지 않음 (password를 가진 다른 클래스에서도 이 필드를 숨기려하면 함수로 분리한 다음 여러 클래스에서 호출해야한다.)
- 표준화 (serialize() 메서드는 모든 이벤트 클래스에 있어야만 한다.)

클래스 데코레이터를 활용한 수정 예시
```python
from datetime import datetime

def hide_field(field) -> str:
    return "민감한 정보"

def format_time(field_timestamp: datetime) -> str:
    return field_timestamp.strftime("%y-%m-%d %H:%M")

def show_original(event_field):
    return event_field

class EventSerializer:
    def __init__(self, serialization_field: dict) -> None:
        self.serialization_field = serialization_field
    
    def serialize(self, event) -> dict:
        return {
            field: transformation(getattr(evnet, field))
            for field transformation in 
            self.serialization_field.items()
        }

class Serialization:
    def __init__(self, **transformations):
        self.serializer = EventSerializer(transformations)
    
    def __call__(self, event_class):
        def serialize_method(event_instance);
            return self.serializer.serialize(event_instance)
        event_class.erialize = serialize_method
        return event_class

@Serialization(
    username=show_original,
    password=hide_field,
    ip=show_original,
    timestamp=format_time,
)
class LoginEvnet:
    def __init__(self, username, password, ip, timestamp):
        self.username = username
        self.password = password
        self.ip = ip
        self.timestamp = timestamp
```










