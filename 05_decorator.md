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


### 중첩 함수의 데코레이터
데코레이터는 함수를 받아서 함수를 반환하는 함수이다. (고차함수)  
데코레이터를 파라미터에 전달하려면 다른 수준의 간접 참조가 필요한데, 3가지로 나누어보자면  
1. 파라미터를 받아서 내부함수에 전달.
2. 데코레이터가 될 함수.
3. 데코레이팅 결과를 반환하는 함수.  

즉 3단계의 중첩 함수가 필요하다.

예제를 살펴보자.  
아래와 같은 형태로 데코레이터가 주어진다고 가정해보자  
```python
@retry(arg1, arg2, ...)
# <original_function> = retry(arg1, arg2, ...)(<original_function>) 과 동일
```

여기서 원하는 재시도 횟수 외에도 제어하려는 예외유형을 나타낼 수도 있다.
```python
RETRIES_LIMIT = 3
def with_retry(retries_limit=RETRIES_LIMIT, allowed_exceptions=None):
    allowed_exceptions = allowed_exceptions or (ControlledException,)
    def retry(operaion):
        @wraps(operation)
        def wrapped(*args, **kwargs):
            last_raised = None
            for _ in range(retries_limit):
                try:
                    return operation(*args, **kwargs)
                except alllowed_exceptions as e:
                    logger.info(f'retrying {operation} due to {e}')
                    last_raised = e
            raise last_raised
        return wrapped
    return retry
```
위 데코레이터를 사용하는 방법
```python
@with_retry()
def run_operation(task):
    return task.run()

@with_retry(retries_limit=5)
def reun_with_custom_retries_limit(task)
    return task.run()

@with_retry(allowed_execption=(AtrributeError,))
def run_with_custom_exceptions(task):
    return task.run()

@with_retry(
    retries_limit=4, allowed_exceptions=(ZeroDivisionError, AttributeError)
)
def run_with_custom_parameter(task):
    return task.run()
```

다음은 위 방식을 클래스를 이용한 방법이다
```python
class WithRetry:

    def __init__(self, retryies_limit=RETRIES_LIMIS, allowd_exceptions=None):
        self.retries_limit = retries_limit
        self.allowed_exceptions = allowd_exception or (ControlledException,)

    def __call___(self, operation):
        @wrap(operation)
        ef wrapped(*args, **kwargs):
            last_raised = None
            for _ in range(retries_limit):
                try:
                    return operation(*args, **kwargs)
                except alllowed_exceptions as e:
                    logger.info(f'retrying {operation} due to {e}')
                    last_raised = e
            raise last_raised
        return wrapped

@WithRety(retries_limit=5)
def run ....
```

### 데코레이터 활용 우수 사례
데코레이터가 사용되는 예제는 수 없이 많이 있지만 가장 관련성이 높은 몇 가지만 소개한다.
- 파라미터 변환 : 더 멋진 API를 노출 하기 위해 함수의 서명을 변경하는 경우, 이 떄 파라미터가 어떻게 처리되고 변환되는지를 캡슐화하여 숨길 수 있다.
    - DbC원에 따라 사전조건 또는 사후조건을 강제할 수도 있다.
    - 유사한 객체를 반복적으로 생성하거나 추상화를 위해 유사한 변형을 반복하는 경우가 있다.
- 코드 추적 : 파라미터와 함꼐 함수의 실행을 로깅하려는 경우
    - 모니터링 하고자하는 함수의 실행과 관련
    - 실제 함수의 실행 경로 추적
    - 함수 지표 모니터링
    - 함수의 실행 시간 측정
    - 언제 함수가 실행되고 전달된 파라미터는 무엇인지 로깅
- 파라미터 유효성 검사
- 재시도 로직 구현
- 일부 반복 작업으 데코레이터로 이동하여 클래스 단순화

### 데코레이터의 활용 - 실수 피하기
효과적인 데코레이터를 만들기 위해 피해야 할 몇 가지 공통된 사항은 아래와 같다.

#### 래핑된 원본 객체의 데이터 보존
원본 함수의 일부 프로퍼티 또는 속성을 유지하지 않아 원하지 않는 부작용을 유발
```python
def trace_decorator(function):
    def wrapped(*args, **kwargs):
        logger.info(f'{function.__qualname__} 실행')
        return function(*args, **kwargs)
    return wrapped

@trace_decorator
def process_account(account_id):
    """id별 계정 처리"""
    logger.info(f'{account_id} 계정 처리')
```
위 문제점은 docstring 참조시 wrapped에 감싸져있어 원본함수가 아닌 새로운 함수의 내용이 출력된다.  
해당 문제점은 아래의 @wraps 데코레이터를 추가하여 실제로는 function파라미터 함수를 래핑한 것이라고 알려주는 것이다.
```python
def trace_decorator(function):
    @wraps(function) # functools.wraps
    def wrapped(*args, **kwargs):
        logger.info(f'{function.__qualname__} 실행')
        return function(*args, **kwargs)
    return wrapped
```

### 데코레이터 부작용 처리
- 가장 안쪽에 정의된 함수여야 한다.
아래 함수의 실행시간을 측정하는 데코레이션의 예를 보자 start_time이 안쪽이 아닌 바깥쪽에 위치한걸 주시하자.
```python
def traced_function_wrong(function):
    logger.info(f'{function} 함수 실행')
    start_time = time.time()

    @functools.wraps(function)
    def wrapped(*args, **kwargs):
        result = function(*args, **kwargs)
        logger.info(
            f'함수 {function}의 실행시간: {time.time() - start_time}'
        )
        return result
    return wrapped

@traced_function_wrong
def process_with_delay(callback, delay=0):
    time.sleep(delay)
    return callback()
```
위 코드를 다른 곳에서 import하여 사용하면 문제점이 생긴다.
매번 호출시 실행시간이 점점 늘어난다.  
@traced_function_wrong은 모듈을 임포트 할 떄 실행된다.  
**process_with_delay = traced_function_wrong(process_with_delay)**  
따라서 함수에 설정된 start_time은 모듈을 처음 임포트할때의 시간이되어 함수에 호출에 따라logging 타임이 잘못된 시점을 기록된다.   
그렇기 때문에 해당 코드는 아래 코드처럼 내부로 이동하여 사용한다.
```python
def traced_function_wrong(function):
    @functools.wraps(function)
    def wrapped(*args, **kwargs):
        logger.info(f'{function} 함수 실행')
        start_time = time.time()
        result = function(*args, **kwargs)
        logger.info(
            f'함수 {function}의 실행시간: {time.time() - start_time}'
        )
        return result
    return wrapped
```













