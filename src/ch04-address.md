# 4.3. Вычисление адреса

Преобразование `(segment, bus, device, function, offset)` в адрес
памяти — центральная операция драйвера.

## Формула

```rust
pub fn offset_in_window(bus: u8, device: u8, function: u8, off: u16) -> usize {
    ((bus as usize) << 20)
        | ((device as usize & 0x1F) << 15)
        | ((function as usize & 0x07) << 12)
        | (off as usize & 0xFFF)
}
```

## С учётом начала окна

```rust
pub fn resolve(window: &EcamWindow, a: Address, off: u16, width: usize)
    -> Result<usize, Error>
{
    if a.segment != window.segment {
        return Err(Error::WrongSegment);
    }
    if a.bus < window.start_bus || a.bus > window.end_bus {
        return Err(Error::BusOutOfRange);
    }
    if a.device > 31 || a.function > 7 {
        return Err(Error::BadAddress);
    }
    if off as usize + width > 0x1000 {
        return Err(Error::OutOfRange);
    }

    let rel = a.bus.wrapping_sub(window.start_bus) as usize;
    let base = window.base
        + (rel << 20)
        + ((a.device as usize) << 15)
        + ((a.function as usize) << 12)
        + (off as usize);

    Ok(base)
}
```

## Порядок проверок

| Шаг | Проверка | Почему |
|-----|----------|--------|
| 1 | Сегмент | Окно принадлежит одному сегменту |
| 2 | Шина в диапазоне | Не выходить за описанное окно |
| 3 | Устройство и функция | Уложиться в 5 и 3 бита |
| 4 | Смещение и ширина | Не выйти за 4 КБ |
| 5 | Выравнивание | Требование платформы |

Дешёвые проверки идут первыми, чтобы не вычислять адрес зря.

## Переполнение

При `usize` = 32 бита адрес окна плюс 256 МБ может переполниться.
Проверка:

```rust
let addr = window.base.checked_add(delta).ok_or(Error::Overflow)?;
```

На 64-битных платформах переполнение маловероятно, но проверка
не мешает.

## Несколько окон

```rust
fn resolve(&self, a: Address, off: u16, width: usize) -> Result<usize, Error> {
    for w in &self.windows[..self.count] {
        if w.segment == a.segment
            && a.bus >= w.start_bus
            && a.bus <= w.end_bus
        {
            return resolve(w, a, off, width);
        }
    }
    Err(Error::NoWindow)
}
```

Линейный поиск по небольшому массиву быстрее, чем хеш-таблица,
и не требует `alloc`.

## Проверка выравнивания

```rust
if (off as usize) % width != 0 {
    return Err(Error::Misaligned);
}
```

Для width = 1 проверка всегда проходит. Для 2 и 4 — отсекает
некорректные обращения до записи в память.

## Кэш вычислений

Если перечисление обращается к одной функции много раз, полезно
хранить вычисленную базу функции:

```rust
fn function_base(&self, a: Address) -> Result<usize, Error> {
    self.resolve(a, 0, 1)
}
```

Тогда чтение регистра — это `base + offset` без повторной
проверки сегмента и шины.

## Ошибки адресации

| Ошибка | Причина |
|--------|---------|
| `NoWindow` | Нет окна для сегмента |
| `BusOutOfRange` | Шина вне описанного диапазона |
| `WrongSegment` | Окно другого сегмента |
| `OutOfRange` | Смещение + ширина больше 4 КБ |
| `Misaligned` | Адрес не кратен разрядности |
| `Overflow` | Переполнение при сложении |
