# 4.9. Размещение BAR

Размещение ресурсов — самая заметная часть работы драйвера,
потому что ошибки здесь проявляются как «устройство не отвечает».

## Алгоритм

```text
1. Собрать запросы: (адрес функции, номер BAR, размер, тип).
2. Определить окна платформы для памяти и ввода-вывода.
3. Разместить запросы от большего к меньшему.
4. Записать адреса в BAR.
5. Обновить окна мостов.
6. Включить декодирование.
```

## Запрос

```rust
pub struct BarRequest {
    pub addr: Address,
    pub bar: u8,
    pub size: u64,
    pub is_io: bool,
    pub prefetchable: bool,
    pub is_64: bool,
}
```

## Сбор

```rust
fn probe_bars(cfg: &impl ConfigSpace, a: Address) -> Result<(), Error> {
    let mut bar = 0u8;

    while bar < 6 {
        let off = 0x10 + 4 * bar as u16;
        let orig = cfg.read32(a, off)?;

        if orig == 0 {
            bar += 1;
            continue;
        }
        if orig & 1 == 1 {
            // окно ввода-вывода
            let size = measure_io_bar(cfg, a, bar)?;
            record_bar(a, bar, size, true, false, false)?;
            bar += 1;
        } else {
            let is_64 = (orig >> 1) & 3 == 2;
            let prefetchable = (orig >> 3) & 1 == 1;
            let size = if is_64 {
                measure_bar64(cfg, a, bar)?
            } else {
                measure_bar32(cfg, a, bar)?
            };

            record_bar(a, bar, size, false, prefetchable, is_64)?;
            bar += if is_64 { 2 } else { 1 };
        }
    }
    Ok(())
}
```

## Измерение

```rust
fn measure_bar32(cfg: &impl ConfigSpace, a: Address, bar: u8) -> Result<u64, Error> {
    let off = 0x10 + 4 * bar as u16;
    let orig = cfg.read32(a, off)?;

    cfg.write32(a, off, 0xFFFF_FFFF)?;
    let mask = cfg.read32(a, off)?;
    cfg.write32(a, off, orig)?;

    let size = (!(mask & !0xF)).wrapping_add(1) as u64;
    Ok(size)
}
```

Для 64-битного BAR измерение охватывает два слова:

```text
mask64 = (read32(off + 4) as u64) << 32 | read32(off)
size   = (!mask64) + 1
```

## Выделение адреса

```rust
pub struct Region {
    pub start: u64,
    pub end: u64,
    pub next: u64,
}

impl Region {
    pub fn alloc(&mut self, size: u64, align: u64) -> Option<u64> {
        let aligned = (self.next + align - 1) & !(align - 1);
        if aligned + size > self.end {
            return None;
        }
        self.next = aligned + size;
        Some(aligned)
    }
}
```

## Выравнивание

| Тип окна | Выравнивание |
|----------|--------------|
| Память 32-бит | Размер окна |
| Память 64-бит | Размер окна |
| Ввод-вывод | Размер окна |
| Prefetchable | Размер окна или больше |

Если окно 1 МБ, оно должно начинаться с адреса, кратного 1 МБ.

## Запись

```rust
pub fn assign(
    cfg: &impl ConfigSpace,
    a: Address,
    r: &BarRequest,
    addr: u64,
) -> Result<(), Error> {
    let off = 0x10 + 4 * r.bar as u16;

    if r.is_64 {
        cfg.write32(a, off, addr as u32)?;
        cfg.write32(a, off + 4, (addr >> 32) as u32)?;
    } else {
        cfg.write32(a, off, addr as u32)?;
    }
    Ok(())
}
```

## Окна мостов

После размещения всех потомков:

```rust
let mut mem_lo = u64::MAX;
let mut mem_hi = 0u64;

for d in descendants(bridge) {
    for bar in d.bars {
        mem_lo = mem_lo.min(bar.addr);
        mem_hi = mem_hi.max(bar.addr + bar.size);
    }
}

cfg.write16(bridge, 0x20, (mem_lo >> 16) as u16)?;
cfg.write16(bridge, 0x22, (mem_hi.saturating_sub(1) >> 16) as u16)?;
```

Младшие биты полей игнорируются аппаратурой, поэтому запись
округлённых значений допустима.

## Включение

```rust
let mut cmd = cfg.read16(a, 0x04)?;
cmd |= 0x0002;                 // Memory Space
if needs_dma { cmd |= 0x0004; } // Bus Master
cfg.write16(a, 0x04, cmd)?;
fence();
```

## Что проверять после

| Проверка | Как |
|----------|-----|
| Адрес записался | Прочитать BAR |
| Декодирование включено | Прочитать Command |
| Устройство отвечает | Прочитать регистр по MMIO |
| Нет конфликтов | Сравнить диапазоны |

## Типичные ошибки

| Ошибка | Следствие |
|--------|-----------|
| Забыть выравнивание | Устройство молчит |
| Перепутать 32 и 64 бита | Второй BAR затёрт |
| Не обновить окно моста | Транзакции не проходят |
| Включить DMA до BAR | Ошибки шины |
| Пересечение окон | Взаимное затирание |
