# TL

TL (Type Language) - это язык для описания структур данных.

Для структуризации полезных данных, при общении используются [TL схемы](https://github.com/ton-blockchain/ton/tree/master/tl/generate/scheme).

TL оперирует 32 битными блоками. Соответственно размер данных в TL должен быть кратен 4 байтам. 
Если размер объекта не кратен 4, нам нужно добавить нужное количество нулевых байт до кратности. 

Для кодирования чисел всегда используется порядок Little Endian.

Более детально TL можно изучить в [документации Telegram](https://core.telegram.org/mtproto/TL) (опционально)


### Кодирование bytes в TL
Для кодирования массива байт нам нужно сначала определить его размер. 
Если он меньше чем 254 байта, то используется кодирование с 1 байтом в качестве размера. Если больше, 
то в качестве первого байта пишется 0xFE, как индикатор большого массива, и уже после него следуют 3 байта размера.

Например, мы кодируем массив `[0xAA, 0xBB]`, его размер 2. Мы используем 1 байт 
размера и далее пишем сами данные, получаем `[0x02, 0xAA, 0xBB]`, готово, но мы видим, 
что финальный размер равен 3 и не кратен 4м байтам, тогда нам нужно добавить 1 байт паддинга, чтобы было 4. Получаем: `[0x02, 0xAA, 0xBB, 0x00]`.

В случае, если нам нужно закодировать массив, размер которого будет равен, например, 396, 
мы поступаем следующим образом: 396 >= 254, значит мы используем 3 байта для кодирования размера и 1 байт индикатор повышенного размера, 
получаем: `[0xFE, 0x8C, 0x01, 0x00, байты массива]`, 396+4 = 400, что кратно 4, выравнивать не нужно.

### Неочевидные правила сериализации

Часто перед самой схемой пишется префикс 4 байта - ее ID. ID схемы это CRC32 с таблицей IEEEE от текста схемы, при этом из текста предварительно удаляются такие симолы как `;` и скобки `()`. Сериализация схемы с ID префиксом называется **boxed**, это позволяет парсеру определить какая схема перед ним, если возможны несколько вариантов.

Как определить, сериализовать в виде boxed или нет? Если наша схема является частью другой схемы, то нужно посмотреть как указан тип поля, если он указан явно, значит сериализуем без префикса, если не явно (есть много таких типов), значит нужно сериализовать в виде boxed. Пример:
```
pub.unenc data:bytes = PublicKey;
pub.ed25519 key:int256 = PublicKey;
pub.aes key:int256 = PublicKey;
pub.overlay name:bytes = PublicKey;
```
У нас есть такие типы, если в схеме указано `PublicKey`, например `adnl.node id:PublicKey addr_list:adnl.addressList = adnl.Node`, значит указано не явно и нам нужно сериализовать с ID префиксом (boxed). А если было бы указано так: `adnl.node id:pub.ed25519 addr_list:adnl.addressList = adnl.Node`, тогда было бы явно, и префикс был бы не нужен.
