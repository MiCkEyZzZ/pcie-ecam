# 4.7. Состояние перечисления

Перечисление — рекурсивный обход, но рекурсия в `no_std`
нежелательна: стек ограничен.

## Итеративный вариант

```rust
pub struct Enumerator {
    stack: [Frame; 16],
    depth: usize,
    segment: u16,
    next_bus: u8,
    max_bus: u8,
}

pub struct Frame {
    pub bus: u8,
    pub device: u8,
    pub function: u8,
    pub last_function: u8,
    pub phase: Phase,
}

pub enum Phase {
    /// Проверка функции 0 и чтение типа заголовка.
    Probe,
    /// Перебор функций устройства.
    Functions,
    /// Запись subordinate после обхода поддерева.
    AfterBridge { bridge: Address },
}
```

## Смысл фаз

| Фаза | Что делает |
|------|------------|
| `Probe` | Читает Vendor ID и Header Type, решает, сколько функций проверять |
| `Functions` | Перебирает функции, для мостов спускается ниже |
| `AfterBridge` | Записывает фактический `subordinate` и возвращается к перебору |

## Шаг обхода

```rust
pub fn step(&mut self, cfg: &impl ConfigSpace)
    -> Result<Option<Address>, Error>
{
    loop {
        let Some(frame) = self.stack.get_mut(self.depth) else {
            return Ok(None);                    // обход закончен
        };

        match frame.phase {
            Phase::Probe => {
                let a = Address {
                    segment: self.segment,
                    bus: frame.bus,
                    device: frame.device,
                    function: 0,
                };

                if cfg.read16(a, 0x00)? == 0xFFFF {
                    self.pop();
                    continue;
                }

                let hdr = cfg.read8(a, 0x0E)?;
                frame.last_function = if hdr & 0x80 != 0 { 7 } else { 0 };
                frame.function = 0;
                frame.phase = Phase::Functions;
            }

            Phase::Functions => {
                let fn_no = frame.function;
                let a = Address {
                    segment: self.segment,
                    bus: frame.bus,
                    device: frame.device,
                    function: fn_no,
                };

                if cfg.read16(a, 0x00)? != 0xFFFF {
                    if cfg.read8(a, 0x0E)? & 0x7F == 0x01 {
                        let sec = self.alloc_bus()?;
                        cfg.write8(a, 0x18, frame.bus)?;
                        cfg.write8(a, 0x19, sec)?;
                        cfg.write8(a, 0x1A, sec)?;

                        frame.phase = Phase::AfterBridge { bridge: a };
                        self.push(sec, 0)?;
                        continue;
                    }
                    return Ok(Some(a));         // обычная функция
                }

                if fn_no >= frame.last_function {
                    self.pop();
                } else {
                    frame.function += 1;
                }
            }

            Phase::AfterBridge { bridge } => {
                let max = self.max_bus;
                let b = *bridge;
                cfg.write8(b, 0x1A, max)?;
                frame.phase = Phase::Functions;
                frame.function += 1;
            }
        }
    }
}
```

## Вспомогательные операции

| Метод | Действие |
|-------|----------|
| `push(bus, device)` | Кладёт новый кадр в фазе `Probe` |
| `pop()` | Убирает верхний кадр |
| `alloc_bus()` | Выдаёт следующий номер, обновляет `max_bus` |

```rust
fn push(&mut self, bus: u8, device: u8) {
    if self.depth + 1 < self.stack.len() {
        self.depth += 1;
        self.stack[self.depth] = Frame {
            bus,
            device,
            function: 0,
            last_function: 0,
            phase: Phase::Probe,
        };
    }
}
```

## Почему `max_bus`

`max_bus` хранит наибольший выданный номер. При возврате из
поддерева он записывается в `Subordinate Bus Number` моста.
Если бы там осталось значение, выданное при спуске, мост
не пропускал бы транзакции к шинам, назначенным позже.

## Размер стека

| Глубина дерева | Кадров |
|----------------|--------|
| Типичная система | 3–6 |
| Максимум в сегменте | 256 |

Массив на 16 кадров покрывает практические конфигурации.
При переполнении обход останавливается с ошибкой, а не портит
стек.

## Список найденных функций

```rust
pub struct DeviceList {
    items: [Device; 64],
    len: usize,
}

pub struct Device {
    pub address: Address,
    pub vendor: u16,
    pub device: u16,
    pub class: u32,
    pub header: u8,
}
```

Фиксированный массив — компромисс: 64 устройства покрывают
большинство систем, а память выделяется статически.

Для больших систем используется callback:

```rust
pub fn walk(
    cfg: &impl ConfigSpace,
    mut on_device: impl FnMut(&Device) -> Result<(), Error>,
) -> Result<(), Error>
```

## Порядок обхода

```text
1. Начать с корневой шины.
2. Проверить устройство 0.
3. Если мост — назначить шину и спуститься.
4. После возврата — подняться и продолжить.
5. Закончить, когда стек пуст.
```

## Что хранится в кадре

| Поле | Зачем |
|------|-------|
| Шина | Текущая шина |
| Устройство | Номер устройства |
| Функция | Номер функции |
| Фаза | Где остановились |
| Мост | Для записи `subordinate` |

Без фазы пришлось бы хранить позицию в отдельной переменной,
что усложняет возобновление.

## Прерывание обхода

Обход может быть приостановлен и продолжен: состояние полностью
находится в структуре, без скрытых данных в стеке вызовов. Это
позволяет:

- растянуть перечисление по времени;
- обрабатывать прерывания между шагами;
- тестировать пошагово.
