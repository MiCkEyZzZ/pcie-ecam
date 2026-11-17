# 6.1. Устройство pci-host-generic

В ядре Linux ECAM реализован в `drivers/pci/controller/pci-host-generic.c`
и общей части `drivers/pci/ecam.c`. Разбор полезен как эталон.

## Слои

```text
pci-host-generic.c
  ├─ разбор дерева устройств / ACPI
  ├─ создание окна ECAM
  └─ регистрация host bridge
        │
        ▼
ecam.c
  ├─ struct pci_ecam_ops
  ├─ pci_ecam_map_bus()
  ├─ pci_ecam_read/write
  └─ pci_ecam_create()
        │
        ▼
probe.c
  ├─ обход шин
  ├─ распределение ресурсов
  └─ регистрация устройств
```

## Ключевая структура

```c
struct pci_ecam_ops {
    unsigned int bus_shift;
    struct pci_ops pci_ops;
    int (*init)(struct pci_config_window *);
};
```

| Поле | Назначение |
|------|------------|
| `bus_shift` | Сдвиг для номера шины (обычно 20, но бывает 16) |
| `pci_ops` | Функции чтения и записи |
| `init` | Специфичная инициализация |

`bus_shift = 16` встречается у некоторых контроллеров, где окно
на шину занимает 64 КБ, а не 1 МБ.

## Вычисление адреса

```c
static void __iomem *pci_ecam_map_bus(struct pci_bus *bus,
                                      unsigned int devfn,
                                      int where)
{
    struct pci_config_window *cfg = bus->sysdata;
    unsigned int busn = bus->number;
    void __iomem *base;

    if (!cfg || busn < cfg->busn_start || busn > cfg->busn_end)
        return NULL;

    base = cfg->win + ((busn - cfg->busn_start) << cfg->ops->bus_shift);
    base += (PCI_SLOT(devfn) << 15) + (PCI_FUNC(devfn) << 12);

    return base + (where & ~3);
}
```

## Общая функция доступа

```c
static int pci_ecam_read(struct pci_bus *bus, unsigned int devfn,
                         int where, int size, u32 *val)
{
    void __iomem *addr = pci_ecam_map_bus(bus, devfn, where);

    switch (size) {
    case 1: *val = readb(addr + (where & 3)); break;
    case 2: *val = readw(addr + (where & 2)); break;
    case 4: *val = readl(addr); break;
    default: return PCIBIOS_BAD_REGISTER_NUMBER;
    }
    return PCIBIOS_SUCCESSFUL;
}
```

Функции `readb`/`readw`/`readl` уже содержат нужные барьеры
и работу с `volatile`.

## Создание окна

```c
struct pci_config_window *pci_ecam_create(struct device *dev,
                                          struct resource *cfgres,
                                          struct resource *busr,
                                          const struct pci_ecam_ops *ops)
{
    /* отображение cfgres в память */
    /* заполнение win, busn_start, busn_end */
    /* возврат структуры */
}
```

## Разбор дерева устройств

```c
static int gen_pci_parse_bus_range(struct device_node *node,
                                   struct resource *res)
{
    const __be32 *range;
    int len;

    range = of_get_property(node, "bus-range", &len);
    if (!range || len < 2 * sizeof(*range))
        return -EINVAL;

    res->start = be32_to_cpup(range);
    res->end   = be32_to_cpup(range + 1);
    return 0;
}
```

## Что делает `pci_host_common_probe`

| Шаг | Действие |
|-----|----------|
| 1 | Найти узел `pci-host-ecam-generic` |
| 2 | Прочитать `reg` и `bus-range` |
| 3 | Разобрать `ranges` в окна ресурсов |
| 4 | Создать окно ECAM |
| 5 | Зарегистрировать `pci_host_bridge` |
| 6 | Вызвать `pci_host_probe` |

## Перечисление

Перечисление выполняет общий код:

```c
pci_host_probe(bridge);
  ├─ pci_scan_root_bus_bridge()
  │    ├─ pci_scan_child_bus()
  │    └─ pci_bus_assign_resources()
  └─ pci_bus_add_devices()
```

| Функция | Задача |
|---------|--------|
| `pci_scan_child_bus` | Обход шин и создание `pci_dev` |
| `pci_bus_assign_resources` | Выделение адресов |
| `pci_bus_add_devices` | Привязка драйверов |

## Обработка ошибок

| Место | Поведение |
|-------|-----------|
| Нет `reg` | `-EINVAL`, мост не создаётся |
| Некорректный `bus-range` | `-EINVAL` |
| Окно занято | `-EBUSY` |
| Нет устройств | Мост создаётся, список пуст |

## Отличия от минимального драйвера

| Возможность | В Linux |
|-------------|---------|
| Несколько окон | Да, список ресурсов |
| Сдвиг шины 16 или 20 | Параметр `bus_shift` |
| ACPI и DT | Оба пути |
| Горячее подключение | Да |
| Управление питанием | Да |

## Что взять для себя

1. Разделение «окно» и «доступ» на два слоя.
2. Параметр сдвига шины вместо жёсткой константы.
3. Общая функция перечисления, не зависящая от способа доступа.
4. Явная проверка границ `bus-range`.

## Что не копировать

| Решение | Почему |
|---------|--------|
| Динамическая память под устройства | В `no_std` недоступна |
| Сложная модель ресурсов | Избыточна для небольшого ядра |
| Абстракции `struct device` | Требуют инфраструктуры ядра |

## Вывод

Эталонная реализация показывает, что ECAM-драйвер — это
небольшой слой доступа плюс общий алгоритм перечисления.
Сложность сосредоточена не в самом доступе, а в распределении
ресурсов и обработке особенностей платформы.
