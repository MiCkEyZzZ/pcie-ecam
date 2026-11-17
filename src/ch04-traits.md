# 4.2. Абстракция доступа к конфигурации

Единый интерфейс позволяет перечислению не знать, каким способом
получен доступ — через ECAM, через порты или через эмуляцию.

## Трейт

```rust
pub trait ConfigSpace {
    fn read8(&self, a: Address, off: u16) -> Result<u8, Error>;
    fn read16(&self, a: Address, off: u16) -> Result<u16, Error>;
    fn read32(&self, a: Address, off: u16) -> Result<u32, Error>;

    fn write8(&self, a: Address, off: u16, v: u8) -> Result<(), Error>;
    fn write16(&self, a: Address, off: u16, v: u16) -> Result<(), Error>;
    fn write32(&self, a: Address, off: u16, v: u32) -> Result<(), Error>;
}
```

## Почему `&self`, а не `&mut self`

Доступ к конфигурации — это операция над внешним устройством,
а не изменение локального состояния. Внутренние поля (список окон)
не меняются, поэтому достаточно разделяемой ссылки. Изменяемость
обеспечивается `volatile`-записями.

## Реализация для ECAM

```rust
pub struct Ecam {
    windows: [EcamWindow; MAX_WINDOWS],
    count: usize,
    lock: SpinLock,
}

impl ConfigSpace for Ecam {
    fn read32(&self, a: Address, off: u16) -> Result<u32, Error> {
        let addr = self.resolve(a, off, 4)?;
        // SAFETY: адрес проверен по границам окна
        Ok(unsafe { core::ptr::read_volatile(addr as *const u32) })
    }

    fn write32(&self, a: Address, off: u16, v: u32) -> Result<(), Error> {
        let addr = self.resolve(a, off, 4)?;
        // SAFETY: см. выше
        unsafe { core::ptr::write_volatile(addr as *mut u32, v) };
        Ok(())
    }
}
```

## Частичные обращения

Младшие разрядности реализуются через чтение слова и маску:

```rust
fn read16(&self, a: Address, off: u16) -> Result<u16, Error> {
    let word = self.read32(a, off & !3)?;
    Ok(((word >> ((off & 2) * 8)) & 0xFFFF) as u16)
}
```

Плюс: работает даже там, где поддерживаются только 32-битные
обращения. Минус: чтение может иметь побочный эффект, а запись
требует чтения-модификации-записи.

| Свойство | Прямой доступ | Через слово |
|----------|---------------|-------------|
| Побочные эффекты | Только у нужного регистра | У соседнего тоже |
| Атомарность | Лучше | Хуже |
| Совместимость | Нужна поддержка | Универсально |

## Альтернативные реализации

| Реализация | Когда нужна |
|------------|-------------|
| `Ecam` | Основной путь |
| `LegacyCf8` | Старые x86 без MCFG |
| `Emulated` | Тесты без оборудования |
| `Filtered` | Ограничение доступа для гостя |

## Тестовая реализация

```rust
pub struct FakeConfig {
    data: [[u8; 256]; 4],
}

impl ConfigSpace for FakeConfig {
    fn read8(&self, a: Address, off: u16) -> Result<u8, Error> {
        Ok(self.data[a.device as usize][off as usize])
    }
    // ...
}
```

Тесты проверяют логику перечисления без реальной шины: это
быстрее и не требует оборудования.

## Ограничение областей

Для гипервизора полезно ограничить доступ:

```rust
pub struct Restricted<'a, C: ConfigSpace> {
    inner: &'a C,
    allowed: &'a [Address],
}

impl<'a, C: ConfigSpace> ConfigSpace for Restricted<'a, C> {
    fn read32(&self, a: Address, off: u16) -> Result<u32, Error> {
        if !self.allowed.contains(&a) {
            return Err(Error::Forbidden);
        }
        self.inner.read32(a, off)
    }
    // ...
}
```

## Композиция

Трейт позволяет складывать поведения: кэш только для
идентификационных регистров, журналирование, ограничение
области. Перечисление при этом не меняется.
