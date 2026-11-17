# 2.7. Заголовки функций и мостов

Тип заголовка определяет смысл байтов `0x10`–`0x3F`.

## Тип 0: обычная функция

```text
0x00  Vendor ID      │ Device ID
0x04  Command        │ Status
0x08  Revision │ Prog IF │ Subclass │ Class
0x0C  Cache │ Latency │ Header │ BIST
0x10  BAR0
0x14  BAR1
0x18  BAR2
0x1C  BAR3
0x20  BAR4
0x24  BAR5
0x28  CardBus CIS Pointer
0x2C  Subsystem Vendor ID │ Subsystem ID
0x30  Expansion ROM Base
0x34  Capabilities Pointer
0x38  Reserved
0x3C  Interrupt Line │ Interrupt Pin │ Min Grant │ Max Latency
```

## Тип 1: мост

```text
0x00  Vendor ID      │ Device ID
0x04  Command        │ Status
0x08  Revision │ Prog IF │ Subclass │ Class
0x0C  Cache │ Latency │ Header │ BIST
0x10  BAR0
0x14  BAR1
0x18  Primary Bus │ Secondary Bus │ Subordinate Bus │ Sec. Latency
0x1C  IO Base │ IO Limit │ Secondary Status
0x20  Memory Base │ Memory Limit
0x24  Prefetchable Base │ Prefetchable Limit
0x28  Prefetchable Base Upper 32
0x2C  Prefetchable Limit Upper 32
0x30  IO Base Upper 16 │ IO Limit Upper 16
0x34  Capabilities Pointer
0x38  Expansion ROM Base
0x3C  Interrupt Line │ Interrupt Pin │ Bridge Control
```

## Ключевое различие

| Поле | Тип 0 | Тип 1 |
|------|-------|-------|
| `0x18` | BAR2 | Номера шин |
| `0x1C` | BAR3 | Окна ввода-вывода |
| `0x20` | BAR4 | Окна памяти |
| `0x24` | BAR5 | Окна prefetchable |
| `0x28` | CardBus CIS | Старшие 32 бита prefetchable base |
| `0x2C` | Subsystem ID | Старшие 32 бита prefetchable limit |
| `0x30` | ROM | Старшие 16 бит окон ввода-вывода |
| `0x3E` | — | Bridge Control |

## Почему это важно при перечислении

Мост — единственный тип, который продолжает дерево. Только у него
есть вторичная шина, которую нужно пронумеровать.

```rust
match header_type & 0x7F {
    0x00 => handle_endpoint(addr),
    0x01 => handle_bridge(addr),
    0x02 => handle_cardbus(addr),
    _    => { /* неизвестный тип, пропустить */ }
}
```

## Мосты CardBus

Тип 2 встречается редко и в современных системах практически
не используется. Его поля похожи на тип 1, но окна описываются
иначе. Для PCIe-драйвера достаточно не падать на нём.

## Subsystem ID

У типа 0 поле `0x2C` содержит идентификаторы подсистемы:
производителя платы и самой платы. По ним драйверы различают
модификации одного и того же контроллера.

| Поле | Смысл |
|------|-------|
| Subsystem Vendor ID | Производитель платы |
| Subsystem ID | Модель платы |

## Что делать с неизвестным типом

| Ситуация | Действие |
|----------|----------|
| Тип `0x02` | Пропустить, не считать мостом PCIe |
| Тип больше `0x02` | Пропустить с записью в лог |
| Тип `0x7F` | Зарезервировано, пропустить |

Неизвестный заголовок не должен приводить к остановке
перечисления: на шине могут быть устройства с нестандартными
расширениями.
