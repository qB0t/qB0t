---
title: Документация Revo API Factoring

language_tabs: # must be one of https://git.io/vQNgJ
  - ruby: Ruby
  - java: Java

toc_footers:
-  <a href='https://revo.ru/'>Revo.ru</a>

#includes:
#  - errors

search: true
---

# Введение

API Factoring реализовано на протоколе HTTPS на основе JSON запросов.

# Авторизация

## Базовые URL адреса

```javascript
BASE_URL = "https://r.revoplus.ru/"
BASE_URL = "https://demo.revoplus.ru/"
```

Запросы по API необходимо посылать по `https`. По запросам, отправленным по http придёт ответ ???. Существует 2 базовых адреса для запросов: production (боевой) и demo (для тестирования). Далее в документации предполагается, что вместо `BASE_URL` будет подставлено одно из значений базовых адресов справа.

## Параметры авторизации

> Пример запроса

```javascript
BASE_URL/factoring/v1/precheck/auth?store_id=123&signature=18e8baf144a924abd1c241d48bd34386
```

Авторизация подразумевает под собой обмен конфиденциальными и опциональными данными между Рево и партнёром. Уникальный идентификатор партнёра `store_id` создается на стороне Рево (в данном примере - число 123). У одного партнера (магазина), может быть несколько уникальных идентификаторов. Секретный ключ `secret_key` используется при формировании электронно-цифровой подписи для аутентификации (проверки подлинности) параметров запроса с целью защиты формы от запуска сторонними лицами. Длина ключа от 8 байт. Алгоритм шифрования SHA1.

## Принцип формирования цифровой подписи

```ruby
require 'digest/sha1'
secret_key = '098f6bcd4621d373cade4e832627b4f6'
data = "{\"callback_url\":\"https://shop.ru/revo/decision\",
\"redirect_url\":\"https://shop.ru/revo/redirect\",
\"current_order\":{\"sum\":\"7500.00\",\"order_id\":\"R001233\"},
\"primary_phone\":\"9268180621\"}"
signature = Digest::SHA1.hexdigest(data + secret_key)
```

```java
import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Formatter;

public class Main {

    static String secret_key = "098f6bcd4621d373cade4e832627b4f6"; // Это пример
    static String data = "{\"callback_url\":\"https://shop.ru/revo/decision\",\"redirect_url\":\"https://shop.ru/revo/redirect\",\"current_order\":{\"sum\":\"7500.00\",\"order_id\":\"R001233\"},\"primary_phone\":\"9268180621\"}";

    public static void main(String[] args) {

        String signature = encryptPassword(data + secret_key); // Тут всегда будет 40 символов по SHA1
        System.out.println(signature);
    }

    private static String encryptPassword(String password) {
        String sha1 = "";
        try {
            MessageDigest crypt = MessageDigest.getInstance("SHA-1");
            crypt.reset();
            crypt.update(password.getBytes("UTF-8"));
            sha1 = byteToHex(crypt.digest());
        } catch(NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch(UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return sha1;
    }

    private static String byteToHex(final byte[] hash) {
        Formatter formatter = new Formatter();
        for (byte b : hash) {
            formatter.format("%02x", b);
        }
        String result = formatter.toString();
        formatter.close();
        return result;
    }
}
```

К строке в формате json (которая передается методом `POST` добавляется секретный ключ `secret_key`, который Рево передает партнеру перед началом интеграции. Далее, к получившейся строке применяется алгоритм SHA1. В результате получаем значение параметра signature. При передаче данных в формате json методом `POST` цифровая подпись передается в строке запроса в виде параметра `signature`. Необходимо обратить внимание, что при формировании signature всегда будет 40 символов по SHA1.

# Методы API

## Registration
<font color="green"> POST </font> `BASE_URL/factoring/v1/precheck/auth`

Метод возвращает адрес iFrame с процедурой регистрации/аутентификации пользователя.

### Parameters

> Пример запроса в формате json

```jsonnet
{
  callback_url: "https://shop.ru/revo/decision",
  redirect_url: "https://shop.ru/revo/redirect",
  current_order:
  {
    order_id: "R107356"
  }
}
```

 | |
-:|:-
**callback_url** <br> <font color="#939da3">string</font>	| URL для ответа от Рево по решению для клиента.
**redirect_url** <br> <font color="#939da3">string</font>	| URL для редиректа после нажатия на кнопку/ссылку в форме Рево "Вернуться в интернет магазин".
**current_order** <br> <font color="#939da3">object</font> | Объект, содержащий информацию о заказе.
**order_id** <br> <font color="#939da3">string, *optional*</font> | Уникальный номер заказа. Не более 255 символов.

В качестве `redirect_url` может выступать страница корзины. Можно вводить дополнительные проверки и перенаправлять пользователя на другие страницы в зависимости от ответа, полученного ранее на `callback_url`.

### Response Parameters

> Пример ответа в формате json

```jsonnet
{
  status: 0,
  message: "Payload valid",
  iframe_url: "https://r.revoplus.ru/form/v1/af45ef12f4233f"
}
```

 | |
-:|:-
**status** <br> <font color="#939da3">integer</font> | 	Код ответа.
**message** <br> <font color="#939da3">string</font> | 	Короткое текстовое описание ответа.
**iframe_url** <br> <font color="#939da3">string</font>	| Cсылка на сгенерированный iFrame.

## Precheck

<font color="green"> POST </font>
`BASE_URL/factoring/v1/precheck/auth?store_id=STORE_ID&signature=YOUR_GENERATED_SHA`

```jsonnet
{
  callback_url: "https://shop.ru/revo/decision",
  redirect_url: "https://shop.ru/revo/redirect",
  current_order:
  {
    amount: "6700.00",
    order_id: "R107356",
    valid_till: "21.04.2017 12:08:01+03:00"
  },
  primary_phone: "8880010203",
  skip_factoring_result: "false"
}
```
Магазин партнера для получения ссылки на iFrame Рево и одновременной аутентификации должен передать три параметра:

store_id - полученный от Рево уникальный идентификатор. (см.подробнее Данный для авторизации)
Служебные данные и данные по клиенту в json(формат json отображен подробнее справа)
Сериализованный в строку исходный json сконкатенированный с секретным ключом. Все это закодировано по SHA1 и является цифровой подписью(см.подробнее Принципе формирования цифровой подписи)
Исходный json c данными отправляются с бэкэнда магазина партнера в теле POST запроса на сервер Рево(см. Базовые URL адреса сервиса)


callback_url – URL куда приходит ответ по кредитному решению Рево для клиента.
redirect_url – URL на который идет редирект после нажатия на кнопку/ссылку в форме Рево "Вернуться в интернет магазин". Можно использовать в качестве redirect_url, например, страницу корзины. Можно также вводить дополнительные проверки и перенаправлять пользователя на другие страницы в зависимости от ответа, полученного ранее на callback_url.
current_order – объект с обязательными полем order_id и amount
amount - сумма в рублях с копейками(важно передавать именно с точкой, т.к это значение типа float).
order_id - номер заказа (уникальное значение, строка до 255 символов). Поле order_id должно быть уникальным в рамках интернет магазина.
valid_till - срок до которого холдирование заказа считается актуальным. Значение параметра обязательно должно включать часовой пояс. После истечения указанного срока, запрос будет отменен.
primary_phone - номер телефона клиента 10 цифр (без кода страны). Важный параметр, который позволяет сразу принимать решения по повторным клиентам не заставляя клиента заполнять длинную форму еще раз.
skip_factoring_result - в данный параметр передаются флаги true/false. По данным флагам определяется, была ли предоплата клиентом до вызова iframe Рево.


Ответы от сервиса Рево


Ответы при кейсах:

Если аутентификация на этапе отправки данных на получение ссылки на iFrame Рево прошла успешно
Если аутентификация не прошла (по какой либо причине)
Описаны статусы ответов и их описание.

## Status

## Finish

## Cancel

## Limit

## Return

# Коды ошибок

Код | Описание
-:|:-
**0**  | Payload valid
**10**  | JSON decode error
**20**  | Order `order_id` missing
**21**  | Wrong order `order_id` format
**22**  | Order exists
**24**  | Order with specified id not found
**30**  | Wrong order `order_sum` format
**40**  | Order `callback_url` missing
**32**  | Order amount is different from the amount specified before
**41**  | Order `redirect_url` missing
**50**  | Store id is missing
**51**  | Store not found
**60**  | `Signature` missing
**61**  | `Signature` wrong
**70**  | Phone number is different
**80**  | Unable to finish - order is already finished/canceled
**81**  | Unable to cancel - order is already finished/canceled
**100** | At the moment the server cannot process your request

# Схемы взаимодействия

Возможны следующие схемы взаимодействия:

<table border="0" align="center">
  <tr>
    <th rowspan="2">Схема</th>
    <th colspan="3" align="center">Предоплата</th>
    <th rowspan="2">Изменение заказа</th>
  </tr>
  <tr>
    <th>Партнёр</th>
    <th>Рево</th>
    <th>Отсутствует</th>
  </tr>
  <tr>
    <td>Факторинг 1</td>
    <td>х</td>
    <td></td>
    <td></td>
    <td>х</td>
  </tr>
  <tr>
    <td>Факторинг 2</td>
    <td></td>
    <td>x</td>
    <td></td>
    <td>х</td>
  </tr>
  <tr>
    <td><a href="http://localhost:4567/#3">Факторинг 3</a></td>
    <td></td>
    <td></td>
    <td>x</td>
    <td>х</td>
  </tr>
  <tr>
    <td>Факторинг 4</td>
    <td>х</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Факторинг 5</td>
    <td></td>
    <td>x</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>Факторинг 6</td>
    <td></td>
    <td></td>
    <td>x</td>
    <td></td>
  </tr>
</table>

## Факторинг 1

## Факторинг 2

## Факторинг 3

Данная схема взаимодействия поддерживает:

- оформление заказа в интернет магазине
- подтверждение или корректировку заказа
- доставку заказа в течении 60 дней после оформления
- частичный или полный возврат заказа

<img src="images/F3.png">

## Факторинг 4

## Факторинг 5

## Факторинг 6
