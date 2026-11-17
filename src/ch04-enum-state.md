# 4.7. Состояние перечисления

Перечисление — рекурсивный обход, но рекурсия в `no_std`
нежелательна: стек ограничен.

## Итеративный вариант

```rust
pub struct Enumerator {
    stack: [Frame; 16],
    depth: usize,
    next_bus: u8,
    max_bus: u8,
}

pub struct Frame {
    pub bus: u8,
    pub device: u8,
    pub function: u8,
    pub phase: Phase,
}

pub enum Phase {
    Start,
    Functions,
    AfterBridge { bridge: Address, first_bus: u8 },
}
```

## Смысл фаз

| Фаза | Что делает |
|------|------------|
| `Start` | Проверяет функцию 0 |
| `Functions` | Перебирает функции |
| `AfterBridge` | Записывает `subordinate` и продолжает |

## Шаг обхода

```rust
pub fn step(&mut self, cfg: &impl ConfigSpace) -> Result<Option<Address>, Error> {
    while let Some(frame) = self.stack.get_mut(self.depth) {
        match frame.phase {
            Phase::Start => {
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
                frame.phase = Phase::Functions;
                self.multi = hdr & 0x80 != 0;
                self.function = 0;
            }
            Phase::Functions => {
                // обход функций
            }
            Phase::AfterBridge { bridge, first_bus } => {
                let max = self.next_bus.saturating_sub(1);
                cfg.write8(bridge, 0x1A, max)?;
                frame.phase = Phase::Functions;
            }
        }
    }
    Ok(None)
}
```

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
