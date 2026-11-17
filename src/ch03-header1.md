# 3.6. Заголовок типа 1

Заголовок моста содержит не окна функции, а окна, которые мост
предоставляет устройствам за собой.

## Идентификация

| Смещение | Размер | Имя |
|----------|--------|-----|
| `0x00` | 2 | Vendor ID |
| `0x02` | 2 | Device ID |
| `0x08` | 1 | Revision ID |
| `0x09` | 1 | Programming Interface |
| `0x0A` | 1 | Subclass |
| `0x0B` | 1 | Class Code |
| `0x0E` | 1 | Header Type |

## Номера шин

| Смещение | Размер | Имя | Тип | Описание |
|----------|--------|-----|-----|----------|
| `0x18` | 1 | Primary Bus | RW | Шина перед мостом |
| `0x19` | 1 | Secondary Bus | RW | Шина за мостом |
| `0x1A` | 1 | Subordinate Bus | RW | Последняя шина поддерева |
| `0x1B` | 1 | Secondary Latency | RW | Таймер вторичной шины |

Правило: `primary <= secondary <= subordinate`.

## Окна ввода-вывода

| Смещение | Размер | Имя |
|----------|--------|-----|
| `0x1C` | 1 | IO Base |
| `0x1D` | 1 | IO Limit |
| `0x30` | 2 | IO Base Upper 16 |
| `0x32` | 2 | IO Limit Upper 16 |

Младшие биты полей `Base` и `Limit` зарезервированы под
выравнивание (по 4 КБ для младших 16 бит).

## Окна памяти

| Смещение | Размер | Имя |
|----------|--------|-----|
| `0x20` | 2 | Memory Base |
| `0x22` | 2 | Memory Limit |
| `0x24` | 2 | Prefetchable Base |
| `0x26` | 2 | Prefetchable Limit |
| `0x28` | 4 | Prefetchable Base Upper 32 |
| `0x2C` | 4 | Prefetchable Limit Upper 32 |

Младшие 4 бита зарезервированы: окно выравнивается на 1 МБ
(для 32-битных полей).

## Управление мостом

| Смещение | Размер | Имя |
|----------|--------|-----|
| `0x3E` | 2 | Bridge Control |

Биты `Bridge Control`:

| Бит | Значение |
|-----|----------|
| 0 | Parity Error Response |
| 1 | SERR# Enable |
| 2 | ISA Enable |
| 3 | VGA Enable |
| 4 | VGA 16-bit Decode |
| 5 | Master Abort Mode |
| 6 | Secondary Bus Reset |
| 7 | Fast Back-to-Back |
| 8 | Primary Discard Timer |
| 9 | Secondary Discard Timer |
| 10 | Discard Timer SERR# Enable |

## Программирование моста

```rust
pub fn program_bridge(
    cfg: &impl ConfigSpace,
    a: Address,
    primary: u8,
    secondary: u8,
) -> Result<(), Error> {
    cfg.write8(a, 0x18, primary)?;
    cfg.write8(a, 0x19, secondary)?;
    cfg.write8(a, 0x1A, secondary)?;   // временно
    Ok(())
}
```

После обхода поддерева:

```rust
let max_bus = highest_bus_in_subtree;
cfg.write8(a, 0x1A, max_bus)?;
```

## Почему subordinate важен

Мост пропускает транзакции, если номер шины в адресе попадает
в диапазон `secondary..=subordinate`. Если поставить слишком
маленькое значение, устройства за мостом станут недоступны.
Если слишком большое — мост будет отвечать на чужие адреса.

## Окна и потомки

```text
Memory Base  = min(адресов всех устройств поддерева) & ~0xFFFFF
Memory Limit = max(адресов + размер) | 0xFFFFF
```

Для prefetchable окон действует то же правило. Важно, чтобы окно
моста полностью покрывало окна потомков, иначе часть устройств
не будет видна.

## Сброс вторичной шины

Бит 6 `Bridge Control` инициирует сброс шины за мостом. После
сброса:

- устройства теряют настройки BAR;
- номера шин сохраняются;
- требуется повторное перечисление поддерева.

## Замечание

Мост PCIe типа 1 может не иметь ни одного BAR: его задача —
маршрутизация, а не предоставление регистров. Наличие BAR
проверяется так же, как у обычной функции.
