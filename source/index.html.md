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

> Базовые адреса

```javascript
BASE_URL = "https://r.revoplus.ru/"  production
BASE_URL = "https://demo.revoplus.ru/"  demo
```

Существует 2 адреса для обращения к методам API: боевой (production) и для тестирования и отладки. В дальнейшем, при вызове методов API предполагается, что вместо `BASE_URL` будет подставлено одно из значений справа.

## Параметры авторизации

Авторизация подразумевает по собой обмен конфиденциальными и опциональными данными между Рево и Магазинов:

Уникальный идентификатор партнёра `store-id` создается на стороне Revo. В данном примере - это число 12. У одного партнера (магазина), может быть несколько уникальных идентификаторов.
Секретный ключ `secret_key` используется при формировании электронно-цифровой подписи для аутентификации (проверки подлинности) параметров запроса с целью защиты формы от запуска сторонними лицами. Длина ключа от 8 байт. Алгоритм шифрования SHA1.

> Пример запроса

```javascript
store_id = 12­
secret_key = "SECRET_KEY_STORE"
```

## Принцип формирования цифровой подписи

К строке в формате json (которая передается методом `POST` добавляется секретный ключ `secret_key`, который Рево передает партнеру перед началом интеграции. Далее, к получившейся строке применяется алгоритм SHA1. В результате получаем значение параметра signature. При передаче данных в формате json методом `POST` цифровая подпись передается в строке запроса в виде параметра `signature`. Необходимо обратить внимание, что при формировании signature всегда будет 40 символов по SHA1.

```ruby
require 'digest/sha1'
secret_key = '098f6bcd4621d373cade4e832627b4f6'
data = "{\"callback_url\":\"https://shop.ru/revo/decision\",\"redirect_url\":\"https://shop.ru/revo/redirect\",\"current_order\":{\"sum\":\"7500.00\",\"order_id\":\"R001233\"},\"primary_phone\":\"9268180621\"}"
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

# Методы API

## Registration
<font color="green"> POST </font> `BASE_URL/factoring/v1/precheck/auth?store_id=STORE_ID&signature=YOUR_GENERATED_SHA`

Магазин партнера для получения ссылки на iFrame Рево с процедурой регистрации/аутентификации пользователя вызывает метод `auth`.

### Параметры пути
Параметр | Описание
-:|:-
**store_id** <br> <font color="#939da3">число</font> | Полученный от Рево [уникальный идентификатор](http://localhost:4567/#3b61794258)
**signature** <br> <font color="#939da3">строка</font> | Цифровая подпись в виде сериализованного в строку json запроса, сконкатенированного с секретным ключом и закодированного по SHA1. Подробнее: [Принцип формирования цифровой подписи] (http://localhost:4567/#1c37860b3b).

> Пример запроса

```jsonnet
https://r.revoplus.ru/factoring/v1/precheck/auth?store_id=12345&signature=adlskdkjf12oasjd14fjasdkflkfjosdafd

{
 "callback_url": "https://shop.ru/revo/decision",
 "redirect_url": "https://shop.ru/revo/redirect",
 "current_order": {
 "order_id": "R107356"
}
```

### Параметры запроса
Параметр | Описание
-:|:-
**callback_url** <br> <font color="#939da3">строка</font>	| URL для ответа от Рево по решению для клиента
**redirect_url** <br> <font color="#939da3">строка</font>	| URL для редиректа после нажатия на кнопку/ссылку в форме Рево "Вернуться в интернет магазин". Например, это может быть страница корзины. Можно также вводить дополнительные проверки и перенаправлять пользователя на другие страницы в зависимости от ответа, полученного ранее на `callback_url`.
**current_order** <br> <font color="#939da3">объект</font> | Объект, содержащий информацию о заказе
**order_id** <br> <font color="#939da3">строка, *опц.*</font> |Уникальный номер заказа. Не более 255 символов

### Параметры ответа

> Пример ответа

```jsonnet
{
  status: 0,
  message: "Payload valid",
  iframe_url: "https://r.revoplus.ru/form/v1/af45ef12f4233f"
}
```

Параметр | Описание
-:|:-
**status** <br> <font color="#939da3">число</font> | 	Код ответа
**message** <br> <font color="#939da3">строка</font> | 	Короткое текстовое описание ответа
**iframe_url** <br> <font color="#939da3">строка</font>	| Cсылка на сгенерированный iFrame

## Precheck

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
