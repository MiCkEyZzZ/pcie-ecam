# 3.5. Список возможностей

Возможности — расширяемые блоки регистров, на которые указывает
поле `Capabilities Pointer` (`0x34`).

## Структура

Каждая запись начинается с двух байтов:

| Смещение | Размер | Поле |
|----------|--------|------|
| `+0` | 1 | Capability ID |
| `+1` | 1 | Next Pointer |

Конец списка — `Next = 0x00`.

```text
0x34 ─▶ 0x40: ID=0x01, next=0x50 (Power Management)
        0x50: ID=0x05, next=0x60 (MSI)
        0x60: ID=0x10, next=0x70 (PCI Express)
        0x70: ID=0x11, next=0x00 (MSI-X)
```

## Основные идентификаторы

| ID | Возможность | Назначение |
|----|-------------|------------|
| `0x01` | Power Management | Управление питанием |
| `0x04` | Slot Identification | Идентификация слота |
| `0x05` | MSI | Прерывания MSI |
| `0x0D` | Subsystem Vendor ID | Идентификаторы подсистемы |
| `0x10` | PCI Express | Параметры линии, управление |
| `0x11` | MSI-X | Расширенные прерывания |

## Перебор

```rust
pub fn for_each_cap(
    cfg: &impl ConfigSpace,
    a: Address,
    mut f: impl FnMut(u8, u8) -> bool,
) -> Result<(), Error> {
    let status = cfg.read16(a, 0x06)?;
    if status & 0x0010 == 0 {
        return Ok(());          // списка нет
    }

    let mut ptr = cfg.read8(a, 0x34)? & 0xFC;
    let mut guard = 0;

    while ptr != 0 && guard < 48 {
        let id = cfg.read8(a, ptr)?;
        let next = cfg.read8(a, ptr + 1)?;

        if !f(ptr, id) {
            break;
        }
        if next == ptr {
            break;              // защита от петли
        }

        ptr = next & 0xFC;
        guard += 1;
    }
    Ok(())
}
```

## Защита от ошибок

| Проблема | Мера |
|----------|------|
| Петля в списке | Счётчик итераций |
| Ссылка на невыровненный адрес | Маска `& 0xFC` |
| Выход за пределы 256 байт | Проверка `ptr + 1 < 0x100` |
| Нет бита Capabilities List | Сразу выйти |

## Возможность PCI Express (`0x10`)

Содержит параметры линии и управление:

| Смещение | Поле |
|----------|------|
| `+0x00` | Capability ID / Next / версия |
| `+0x02` | Device Capabilities |
| `+0x04` | Device Control |
| `+0x06` | Device Status |
| `+0x08` | Link Capabilities |
| `+0x0A` | Link Control |
| `+0x0C` | Link Status |
| `+0x10` | Slot Capabilities |

Полезные поля:

| Поле | Зачем |
|------|-------|
| Max Payload Size | Размер полезной нагрузки TLP |
| Max Read Request | Размер запроса чтения |
| Link Speed | Скорость линии |
| Link Width | Число линий |
| Link Training | Состояние тренировки |

## MSI (`0x05`)

Состав зависит от разрядности адреса (бит 7 в `Message Control`):

| Смещение | 32-битный адрес | 64-битный адрес |
|----------|-----------------|-----------------|
| `+0x00` | ID / Next / Message Control | то же |
| `+0x04` | Message Address | Message Address, младшие 32 |
| `+0x08` | Message Data | Message Address, старшие 32 |
| `+0x0C` | Mask Bits (если есть) | Message Data |
| `+0x10` | Pending Bits | Mask Bits (если есть) |
| `+0x14` | — | Pending Bits |

Поле `Message Data` задаёт вектор; адрес указывает в пространство
контроллера прерываний.

## MSI-X (`0x11`)

| Смещение | Поле |
|----------|------|
| `+0x00` | ID / Next / Message Control |
| `+0x04` | Table Offset / BIR |
| `+0x08` | PBA Offset / BIR |

Таблица MSI-X лежит в BAR, указанном в `BIR`, по смещению
из `Table Offset`. Каждая запись — 16 байт.

## Практическое замечание

Список возможностей может быть невыровненным по 4 байта, но
на практике указатели кратны четырём. Чтение байтовыми
обращениями надёжнее, чем чтение словами, если реализация
не гарантирует доступ к отдельным байтам.

## Порядок использования

```text
1. Проверить бит Capabilities List в Status.
2. Пройти список, собрать смещения нужных возможностей.
3. Работать с найденными блоками по их собственным правилам.
4. Не предполагать порядок возможностей в списке.
```
