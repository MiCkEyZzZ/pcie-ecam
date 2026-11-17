# 5.2. Описание в дереве устройств

На платформах без ACPI окно ECAM описывается узлом
`pci-host-ecam-generic`.

## Минимальный узел

```dts
pcie@10000000 {
    compatible = "pci-host-ecam-generic";
    device_type = "pci";
    reg = <0x00 0x10000000 0x00 0x10000000>;
    bus-range = <0x00 0xff>;
    #address-cells = <3>;
    #size-cells = <2>;
    #interrupt-cells = <1>;
    interrupt-map-mask = <0xf800 0 0 7>;
    interrupt-map = <0 0 0 1 &plic 0x20 4>;
    ranges = <0x01000000 0 0x00000000 0x00 0x3eff0000 0 0x00010000>,
             <0x02000000 0 0x10000000 0x00 0x30000000 0 0x10000000>;
};
```

## Свойства

| Свойство | Обязательно | Смысл |
|----------|-------------|-------|
| `compatible` | Да | `pci-host-ecam-generic` |
| `device_type` | Да | `pci` |
| `reg` | Да | Базовый адрес и размер окна |
| `bus-range` | Нет | Диапазон шин, по умолчанию 0–255 |
| `ranges` | Да | Окна памяти и ввода-вывода |
| `#address-cells` | Да | 3 для PCI |
| `#size-cells` | Да | 2 для PCI |
| `interrupt-map` | Нет | Маршрутизация INTx |
| `msi-parent` | Нет | Устройство MSI |
| `dma-coherent` | Нет | DMA когерентно |
| `linux,pci-domain` | Нет | Номер сегмента |

## Как читается `reg`

```text
reg = <base_hi base_lo size_hi size_lo>
```

В примере выше: база `0x10000000`, размер `0x10000000` (256 МБ).

## Как читается `bus-range`

```text
bus-range = <start end>
```

Если свойства нет, используется `0..255`. Драйвер не должен
сканировать шины за пределами диапазона.

## `ranges`

Три вида записей:

| Тип | Первое слово | Смысл |
|-----|--------------|-------|
| Ввод-вывод | `0x01000000` | Окно портов |
| Память 32 | `0x02000000` | Окно MMIO |
| Память 64 | `0x03000000` | Окно MMIO выше 4 ГБ |

Формат: `(пространство, адрес PCI, адрес CPU, размер)`.

## Чтение из кода

```rust
fn from_fdt(node: &FdtNode) -> Result<EcamWindow, Error> {
    let (base, size) = node.reg().ok_or(Error::BadAddress)?;

    let (start, end) = node
        .property("bus-range")
        .and_then(|p| p.as_bus_range())
        .unwrap_or((0, 0xFF));

    let segment = node
        .property("linux,pci-domain")
        .and_then(|p| p.as_u32())
        .unwrap_or(0) as u16;

    Ok(EcamWindow { base, segment, start_bus: start, end_bus: end })
}
```

## Свойства, влияющие на поведение

| Свойство | Влияние |
|----------|---------|
| `dma-coherent` | Не нужно выполнять операции поддержания когерентности |
| `msi-parent` | Куда направлять MSI |
| `interrupt-map` | Как преобразовать INTx в номер прерывания |
| `linux,pci-domain` | Номер сегмента в терминах Linux |

## Отличие от ACPI

| Аспект | ACPI | Дерево устройств |
|--------|------|------------------|
| Описание | Таблица MCFG | Узел с `reg` |
| Окна памяти | Из `_CRS` | Свойство `ranges` |
| Прерывания | `_PRT` | `interrupt-map` |
| Гибкость | Ниже | Выше |

## Несколько узлов

В системе может быть несколько узлов `pci-host-ecam-generic`:
по одному на корневой комплекс. Каждый описывает своё окно
и свой диапазон шин.

```dts
pcie@10000000 { ... linux,pci-domain = <0>; };
pcie@20000000 { ... linux,pci-domain = <1>; };
```

Драйвер должен собрать все узлы, а не брать первый.

## Проверка

| Проверка | Как |
|----------|-----|
| Окно не пересекает RAM | Сравнить с `/memory` |
| `bus-range` валиден | `start <= end` |
| `reg` выровнен | Кратен 4 КБ |
| `ranges` покрывают BAR | Иначе устройства не заработают |
