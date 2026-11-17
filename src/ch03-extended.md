# 3.7. Расширенное пространство

Расширенное configuration space занимает байты `0x100`–`0xFFF`
и доступно только у функций PCIe.

## Формат записи

```text
0x100 ─┬─ ID (16 бит)
       ├─ Version (4 бита)
       ├─ Next (12 бит)
       └─ данные возможности
```

| Поле | Размер | Смысл |
|------|--------|-------|
| ID | 2 | Идентификатор возможности |
| Version | 1 | Версия структуры |
| Next | 1 | Смещение следующей (12 бит) |
| Data | — | Зависит от ID |

Смещение `Next` задаётся в 4-байтных единицах: реальный адрес
равен `next << 2`. Конец цепочки — `next = 0`.

## Основные возможности

| ID | Имя | Назначение |
|----|-----|------------|
| `0x0001` | AER | Расширенная обработка ошибок |
| `0x0002` | Virtual Channel | Виртуальные каналы |
| `0x0003` | Device Serial Number | Серийный номер |
| `0x0004` | Power Budgeting | Бюджет питания |
| `0x0005` | Root Complex Link Declaration | Объявление связей корневого комплекса |
| `0x0006` | Root Complex Internal Link Control | Управление внутренними связями |
| `0x0007` | Root Complex Event Collector Association | Связь с коллектором событий |
| `0x000B` | Vendor-Specific | Специфичное для производителя |
| `0x000D` | ACS | Изоляция транзакций |
| `0x000E` | ARI | Альтернативная нумерация |
| `0x0010` | SR-IOV | Виртуализация |
| `0x0018` | LTR | Требования задержки |
| `0x001D` | DPC | Контроль загрузки |

## AER

Advanced Error Reporting даёт подробную информацию об ошибках.

| Смещение | Регистр |
|----------|---------|
| `+0x00` | Extended Capability Header |
| `+0x04` | Uncorrectable Error Status |
| `+0x08` | Uncorrectable Error Mask |
| `+0x0C` | Uncorrectable Error Severity |
| `+0x10` | Correctable Error Status |
| `+0x14` | Correctable Error Mask |
| `+0x18` | Correctable Error Capabilities |
| `+0x1C` | Header Log (4 слова) |
| `+0x2C` | Root Error Command |
| `+0x30` | Root Error Status |

Статусы очищаются записью единицы (`RW1C`).

## Device Serial Number

| Смещение | Поле |
|----------|------|
| `+0x00` | Header |
| `+0x04` | Serial Number (младшие 32) |
| `+0x08` | Serial Number (старшие 32) |

Уникальный номер полезен для стабильной идентификации устройства
между перезагрузками.

## SR-IOV

| Смещение | Поле |
|----------|------|
| `+0x00` | Header |
| `+0x04` | SR-IOV Capabilities |
| `+0x08` | SR-IOV Control |
| `+0x0A` | SR-IOV Status |
| `+0x0C` | Initial VFs |
| `+0x0E` | Total VFs |
| `+0x10` | Num VFs |
| `+0x14` | VF Offset / Stride |

Виртуальные функции появляются в configuration space только
после включения SR-IOV и выделения VF.

## Перебор

```rust
pub fn for_each_ext_cap(
    cfg: &impl ConfigSpace,
    a: Address,
    mut f: impl FnMut(u16, u16) -> bool,
) -> Result<(), Error> {
    let mut off = 0x100u16;
    let mut guard = 0;

    while off >= 0x100 && off < 0x1000 && guard < 64 {
        let hdr = cfg.read32(a, off)?;
        let id = (hdr & 0xFFFF) as u16;
        let next = ((hdr >> 20) & 0xFFF) as u16;

        if id == 0 || id == 0xFFFF {
            break;
        }
        if !f(off, id) {
            break;
        }

        off = next << 2;
        guard += 1;
    }
    Ok(())
}
```

## Ограничения

| Ограничение | Следствие |
|-------------|-----------|
| Только PCIe | У устройств PCI расширенной области нет |
| Нужны 4 КБ | Стандартных 256 байт недостаточно |
| Смещение в 4-байтных единицах | Легко ошибиться в 4 раза |
| Возможности могут отсутствовать | Не считать их обязательными |

## Практика

При первом включении устройства обычно достаточно:

1. найти AER и очистить статусы;
2. найти Device Serial Number для идентификации;
3. проверить наличие SR-IOV, если нужна виртуализация;
4. не трогать неизвестные возможности.
