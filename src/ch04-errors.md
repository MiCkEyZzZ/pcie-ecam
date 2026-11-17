# 4.5. Ошибки и их обработка

В `no_std` нет `std::error::Error`, поэтому тип ошибки
определяется вручную.

## Тип ошибки

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Error {
    NoWindow,
    WrongSegment,
    BusOutOfRange,
    OutOfRange,
    Misaligned,
    Overflow,
    BadAddress,
    NoDevice,
    Timeout,
    WriteIgnored,
    NoResources,
    Unsupported,
    Forbidden,
}
```

## Категории

| Категория | Ошибки | Реакция |
|-----------|--------|---------|
| Адресация | `NoWindow`, `BusOutOfRange`, `OutOfRange` | Ошибка вызова |
| Данные | `Misaligned`, `BadAddress` | Ошибка вызова |
| Устройство | `NoDevice`, `Timeout` | Пропустить |
| Конфигурация | `NoResources`, `Unsupported` | Сообщить наверх |
| Доступ | `Forbidden` | Отказ |

## Что делать при перечислении

| Ошибка | Действие |
|--------|----------|
| `NoDevice` | Продолжить со следующей функцией |
| `Timeout` | Записать в лог, продолжить |
| `NoResources` | Прекратить выделение, сохранить уже сделанное |
| `Unsupported` | Пропустить устройство |

Перечисление не должно падать целиком из-за одного плохого
устройства.

## Логирование

```rust
fn scan_slot(&self, a: Address) -> Result<(), Error> {
    match self.cfg.read16(a, 0x00) {
        Ok(0xFFFF) => Ok(()),
        Ok(_) => self.probe(a),
        Err(e) => {
            log::warn!("cfg read {:?}: {:?}", a, e);
            Ok(())
        }
    }
}
```

## Таймауты

Если платформа умеет обнаруживать зависание шины, ошибка
возвращается как `Timeout`. Если не умеет — единственная защита
это ограничение диапазона шин.

```rust
pub fn read16_checked(&self, a: Address, off: u16) -> Result<u16, Error> {
    if a.bus > self.max_bus {
        return Err(Error::BusOutOfRange);
    }
    self.read16(a, off)
}
```

## Восстановление после ошибки шины

| Шаг | Действие |
|-----|----------|
| 1 | Зафиксировать адрес и операцию |
| 2 | Проверить, что окно не перекрыто |
| 3 | При необходимости сбросить вторичную шину |
| 4 | Повторить перечисление поддерева |

Сброс вторичной шины — тяжёлая операция: он затрагивает все
устройства за мостом.

## Отличие ошибки от отсутствия

| Наблюдение | Трактовка |
|------------|-----------|
| `0xFFFF` в Vendor ID | Устройства нет |
| `0x0000` | Вероятно, окно не отображено |
| Ошибка доступа | Есть устройство, но доступ сломан |
| Регулярный таймаут | Проблема платформы |

Смешивать эти случаи нельзя: «нет устройства» — штатная
ситуация, «ошибка доступа» — повод для диагностики.

## Публичный API

```rust
pub trait HostBridge {
    fn config(&self) -> &impl ConfigSpace;
    fn windows(&self) -> &[EcamWindow];
    fn enumerate(&mut self) -> Result<(), Error>;
}
```

Ошибки перечисления возвращаются как `Result`, но частичный
результат сохраняется: устройства, найденные до сбоя, остаются
доступны.

## Текстовые описания

```rust
impl Error {
    pub fn as_str(&self) -> &'static str {
        match self {
            Error::NoWindow => "нет окна ECAM для сегмента",
            Error::BusOutOfRange => "шина вне диапазона окна",
            Error::OutOfRange => "смещение за пределами функции",
            Error::Misaligned => "неверное выравнивание",
            Error::NoDevice => "устройство отсутствует",
            Error::Timeout => "таймаут доступа к конфигурации",
            Error::NoResources => "не хватает ресурсов",
            _ => "ошибка",
        }
    }
}
```

Строки не требуют `alloc` и удобны для логов загрузчика.
