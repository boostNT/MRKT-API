# MRKT-API
Это неофициальная документация по апи [маркета](https://t.me/mrkt), одной из площадок для торговли телеграм нфт подарками.

###### для ленивых рабочий код на питоне [**тут**](#полный-рабочий-код-на-питоне)

### При заходе в приложение, для получения подарков идет пост запрос по этому юрл:

```python
https://api.tgmrkt.io/api/v1/gifts/saling
```

#### Чтобы фильтровать необходимые вам подарки, используется фильтрация в json формате:
```python
json_data = {
    "collectionNames": [],
    "modelNames": [],
    "backdropNames": [],
    "symbolNames": [],
    "ordering": "Price",
    "lowToHigh": True,
    "maxPrice": None,
    "minPrice": None,
    "mintable": None,
    "number": None,
    "count": 20,
    "cursor": '',
    "query": None,
    "promotedFirst": False,
}
```


### Разберём по порядку:
- **collectionNames** - Список названий подарков, которые вы хотите увидеть.
- **modelNames** - Список моделей подарков, которые вы хотите увидеть (выдает результат даже если не выбрано название подарка).
- **backdropNames** - Список фонов подарков, которые вы хотите увидеть.
- **symbolNames** - Список узоров подарков, которые вы хотите увидеть.
- **ordering** - Тип сортировки (_Price_ | _ModelRarity_ | _BackgroundRarity_ | _SymbolRarity_ | None (по времени выставления)).
- **lowToHigh** - По возрастанию ли отображать подарки (зависит от выбранного типа сортировки).
- **maxPrice** - Макс. цена подарка.
- **minPrice** - Мин. цена подарка.
- **mintable** - Можно ли вывести подарок в блокчейн (наверное).
- **number** - Номер подарка.
- **count** - Макс. кол-во подарков, которое вы получите в ответе (лимит - 20).
- **cursor** - "Оффсет", с которого получать подарки. Для первой страницы оставляете его пустой строкой, а если необходимо просматривать страницы далее, то берете его из первого запроса из ключа cursor (его значение может быть None, если в первом ответе вы получили меньше подарков, чем указали в лимите), и вставляете в следующий, и так далее.
- **query** - Неизвестно.
- **promotedFirst** - Неизвестно.


#### Но и этого недостаточно
К сожалению, чтобы как-то успешно взаимодействовать с апи маркета через запросы, нужен токен.

Токен берётся по этому юрл:
```python
https://api.tgmrkt.io/api/v1/auth
```
Отправляя сюда пост запрос с json телом `{"data": init_data}`\
Ну а **init_data** можно получить только если вы **реально зашли через тг аппку**\
Следовательно, юзаем пирограм!

```python
import asyncio
from pyrogram import Client
from pyrogram.raw.functions.messages import RequestAppWebView
from pyrogram.raw.types import InputBotAppShortName, InputUser
from urllib.parse import unquote
from curl_cffi import AsyncSession



api_id = 123123123 # замени с https://my.telegram.org/auth
api_hash = '1rghg34hb3g4bkj3gbg134' # замени с https://my.telegram.org/auth
client = Client('main')


async def get_auth_token():
    async with client:
        bot_entity = await client.get_users('mrkt')
        peer = await client.resolve_peer('mrkt')

        bot = InputUser(user_id=bot_entity.id, access_hash=bot_entity.raw.access_hash)
        bot_app = InputBotAppShortName(bot_id=bot, short_name='app')

        web_view = await client.invoke(
            RequestAppWebView(
                peer=peer,
                app=bot_app,
                platform="android",
            )
        )

        init_data = unquote(web_view.url.split('tgWebAppData=', 1)[1].split('&tgWebAppVersion', 1)[0])

        auth_data = {"data": init_data}
        
        async with AsyncSession() as s:
            r = await s.post(url="https://api.tgmrkt.io/api/v1/auth", json=auth_data)
            rj = r.json()
            if rj:
                token = rj.get('token', None)
            else:
                token = ': ('

    return token


async def main():
    token = await get_auth_token()
    print(token)


loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

Либо вы можете зайти в веб тг, нажать f12, открыть вкладку network, зайти на маркет, найти запрос по `https://api.tgmrkt.io/api/v1/auth`, и в Response взять этот токен.   


   
Время действия токена неизвестно, но точно более суток.


### Полный рабочий код на питоне:
```python
import asyncio
from pyrogram import Client
from pyrogram.raw.functions.messages import RequestAppWebView
from pyrogram.raw.types import InputBotAppShortName, InputUser
from urllib.parse import unquote
from curl_cffi import requests

MARKET_API_URL = 'https://api.tgmrkt.io/api/v1'
api_id = 123123123 # замените на своё с https://my.telegram.org/auth
api_hash = '1rghg34hb3g4bkj3gbg134' # замените на своё с https://my.telegram.org/auth
client = Client('main', api_id, api_hash)


async def main():
    token = await get_auth_token()
    # token = '' или через веб тг


    headers = {'Authorization': token}

    json_data = {
        "collectionNames": ['Lunar Snake'],
        "modelNames": ['Albino'],
        "backdropNames": [],
        "symbolNames": [],
        "ordering": "Price",
        "lowToHigh": True,
        "maxPrice": None,
        "minPrice": None,
        "mintable": None,
        "number": None,
        "count": 20,
        "cursor": '',
        "query": None,
        "promotedFirst": False,
    }
    try:
        r=requests.post(f'{MARKET_API_URL}/gifts/saling', headers=headers, json=json_data)
        rj=r.json()
        gifts = rj['gifts']
        print(gifts)

        # Или если нужна следующая страница:
        # cursor = rj['cursor']
        # json_data.update({'cursor': cursor})
        # r=requests.post(f'{MARKET_API_URL}/gifts/saling', headers=headers, json=json_data)
        # rj=r.json()
        # print(gifts)
    except Exception as e:
        print(e)

    # Код выведет первую страницу с подарками Lunar Snake с моделью Albino

async def get_auth_token():
    async with client:
        bot_entity = await client.get_users('mrkt')
        peer = await client.resolve_peer('mrkt')

        bot = InputUser(user_id=bot_entity.id, access_hash=bot_entity.raw.access_hash)
        bot_app = InputBotAppShortName(bot_id=bot, short_name='app')

        web_view = await client.invoke(
            RequestAppWebView(
                peer=peer,
                app=bot_app,
                platform='android',
            )
        )

        init_data = unquote(web_view.url.split('tgWebAppData=', 1)[1].split('&tgWebAppVersion', 1)[0])

        auth_data = {'data': init_data}
        
        r = requests.post(url='https://api.tgmrkt.io/api/v1/auth', json=auth_data)
        rj = r.json()
        if rj:
            token = rj.get('token', None)
        else:
            token = None

    return token


loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

Тг для связи - [@bxxst](t.me/bxxst)
