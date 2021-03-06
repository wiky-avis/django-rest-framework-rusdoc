# Дросселирование (Регулирование)

> HTTP/1.1 420 Enhance Your Calm
>
> [Twitter API rate limiting response][cite]

Регулирование похоже на [разрешения][permissions] в том, что оно определяет, должен ли запрос быть авторизован. Дроссели (Throttles) указывают на временное состояние и используются для управления скоростью запросов, которые клиенты могут отправлять к API.

Как и в случае с разрешениями, можно использовать несколько дросселей. Ваш API может иметь ограничительный дроссель для неаутентифицированных запросов и менее строгий дроссель для аутентифицированных запросов.

Другой сценарий, в котором вы можете захотеть использовать несколько дросселей, - если вам нужно наложить разные ограничения на разные части API из-за того, что некоторые службы являются особенно ресурсоемкими.

Также можно использовать несколько дросселей, если вы хотите установить как частоту пакетного регулирования, так и постоянную скорость дросселирования. Например, вы можете ограничить пользователя до 60 запросов в минуту и ​​1000 запросов в день.

Дроссели не обязательно относятся только к запросам ограничения скорости. Например, службе хранения может также потребоваться регулирование пропускной способности, а платной службе данных может потребоваться регулирование определенного количества записей, к которым осуществляется доступ.

## Как определяется дросселирование

Как и в случае с разрешениями и проверкой подлинности, регулирование в REST framework всегда определяется как список классов.

Перед запуском основной части представления проверяется каждый дроссель в списке.

Если какая-либо проверка дросселя не удалась, будет возбуждено исключение `exceptions.Throttled`, и основная часть представления не будет запущена.

## Установка политики регулирования

Политика регулирования по умолчанию может быть установлена ​​глобально с помощью параметров `DEFAULT_THROTTLE_CLASSES` и `DEFAULT_THROTTLE_RATES`. Например:

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day'
    }
}
```

Описания скорости, используемые в `DEFAULT_THROTTLE_RATES`, могут включать `second`, `minute`, `hour` или `day` в качестве периода дросселирования.

Вы также можете установить политику регулирования для каждого представления или набора представлений, с использованием представлений на основе классов `APIView`.

```python
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle
from rest_framework.views import APIView

class ExampleView(APIView):
    throttle_classes = [UserRateThrottle]

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

Или, если вы используете декоратор `@api_view` с представлениями на основе функций.

```python
@api_view(['GET'])
@throttle_classes([UserRateThrottle])
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```

## Как идентифицируются клиенты

HTTP-заголовок `X-Forwarded-For` и переменная WSGI `REMOTE_ADDR` используются для однозначной идентификации клиентских IP-адресов для регулирования. Если присутствует заголовок `X-Forwarded-For`, то он будет использоваться, в противном случае будет использоваться значение переменной `REMOTE_ADDR` из среды WSGI.

Если вам нужно строго идентифицировать уникальные IP-адреса клиентов, вам необходимо сначала настроить количество прокси приложений, за которыми работает API, установив параметр `NUM_PROXIES`. Этот параметр должен быть целым числом от нуля или более. Если установлено ненулевое значение, то IP-адрес клиента будет определяться как последний IP-адрес в заголовке `X-Forwarded-For` после того, как любые IP-адреса прокси-сервера приложения были сначала исключены. Если установлено в ноль, тогда значение REMOTE_ADDR всегда будет использоваться как идентифицирующий IP-адрес.

Важно понимать, что если вы настроите параметр `NUM_PROXIES`, то все клиенты, находящиеся за уникальным [NAT'd](https://en.wikipedia.org/wiki/Network_address_translation) шлюзом, будут рассматриваться как один клиент.

Дополнительный контекст о том, как работает заголовок `X-Forwarded-For` и определение IP-адреса удаленного клиента, можно [найти здесь][identify-clients].

## Настройка кеша

Классы дросселей, предоставляемые REST framework, используют бэкэнд кеширования Django. Вы должны убедиться, что вы установили соответствующие [настройки кеша][cache-setting]. Значение по умолчанию для backend `LocMemCache` должно подходить для простых настроек. См. [Кеш-документацию][cache-docs] Django для получения более подробной информации.

Если вам нужно использовать кеш, отличный от `'default'`, вы можете сделать это, создав собственный класс дросселя и установив атрибут `cache`. Например:

```python
from django.core.cache import caches

class CustomAnonRateThrottle(AnonRateThrottle):
    cache = caches['alternate']
```

Вам необходимо также не забыть установить свой собственный класс дроссельной заслонки в ключе настроек `'DEFAULT_THROTTLE_CLASSES'` или с помощью атрибута представления `throttle_classes`.

# Справочник по API

## AnonRateThrottle

`AnonRateThrottle` будет блокировать только неаутентифицированных пользователей. IP-адрес входящего запроса используется для генерации уникального ключа, который нужно ограничить.

Допустимая частота запросов определяется одним из следующих (в порядке объявления).

* Свойство `rate` в классе, которое может быть предоставлено путем переопределения `AnonRateThrottle` и установки свойства.
* Параметр `DEFAULT_THROTTLE_RATES['anon']`.

`AnonRateThrottle` подходит, если вы хотите ограничить скорость запросов из неизвестных источников.

## UserRateThrottle

`UserRateThrottle` будет ограничивать пользователей заданной скоростью запросов через API. Идентификатор пользователя используется для генерации уникального ключа, против которого можно действовать. Запросы, не прошедшие проверку подлинности, будут использовать IP-адрес входящего запроса для генерации уникального ключа, который будет подавляться.

Допустимая частота запросов определяется одним из следующих (в порядке предпочтения).

* Свойство `rate` в классе, которое может быть предоставлено путем переопределения `UserRateThrottle` и установки свойства.
* Параметр `DEFAULT_THROTTLE_RATES['user']`.

API может иметь несколько `UserRateThrottles` одновременно. Для этого переопределите `UserRateThrottle` и установите уникальную «область видимости» для каждого класса.

Например, несколько пользовательских дросселей могут быть реализованы с помощью следующих классов ...

```python
class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'
```

... и следующие настройки.

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'example.throttles.BurstRateThrottle',
        'example.throttles.SustainedRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',
        'sustained': '1000/day'
    }
}
```

`UserRateThrottle` подходит, если вам нужны простые глобальные ограничения скорости для каждого пользователя.

## ScopedRateThrottle

Класс `ScopedRateThrottle` можно использовать для ограничения доступа к определенным частям API. Этот дроссель будет применяться только в том случае, если представление, к которому осуществляется доступ, включает свойство `.throttle_scope`. Затем уникальный дроссельный ключ будет сформирован путем объединения "области видимости" запроса с уникальным идентификатором пользователя или IP-адресом.

Допустимая частота запросов определяется установкой `DEFAULT_THROTTLE_RATES` с использованием ключа из "области видимости" запроса.

Например, учитывая следующие представления ...

```python
class ContactListView(APIView):
    throttle_scope = 'contacts'
    ...

class ContactDetailView(APIView):
    throttle_scope = 'contacts'
    ...

class UploadView(APIView):
    throttle_scope = 'uploads'
    ...
```

... и следующие настройки.

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.ScopedRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'contacts': '1000/day',
        'uploads': '20/day'
    }
}
```

Запросы пользователей к `ContactListView` или `ContactDetailView` будут ограничены в общей сложности 1000 запросами в день. Запросы пользователей к `UploadView` будут ограничены 20 запросами в день.

# Custom throttles

# Пользовательские дроссели

Чтобы создать собственный дроссель, переопределите `BaseThrottle` и реализуйте `.allow_request(self, request, view)`. Метод должен возвращать `True`, если запрос должен быть разрешен, и `False` в противном случае.

При желании вы также можете переопределить метод `.wait()`. Если реализовано, `.wait()` должна возвращать рекомендуемое количество секунд ожидания перед попыткой следующего запроса или `None`. Метод `.wait()` будет вызываться только в том случае, если `.allow_request()` ранее вернул `False`.

Если реализован метод `.wait()` и запрос регулируется, то в ответ будет включен заголовок `Retry-After`.

## Example

Ниже приведен пример дросселирования скорости, который произвольно дросселирует 1 из каждых 10 запросов.

```python
import random

class RandomRateThrottle(throttling.BaseThrottle):
    def allow_request(self, request, view):
        return random.randint(1, 10) != 1
```

[cite]: https://developer.twitter.com/en/docs/basics/rate-limiting
[permissions]: permissions.md
[identifying-clients]: http://oxpedia.org/wiki/index.php?title=AppSuite:Grizzly#Multiple_Proxies_in_front_of_the_cluster
[cache-setting]: https://docs.djangoproject.com/en/stable/ref/settings/#caches
[cache-docs]: https://docs.djangoproject.com/en/stable/topics/cache/#setting-up-the-cache
