# Приложение Б. Глоссарий

| Термин | Расшифровка | Значение |
|--------|-------------|----------|
| **ACPI** | Advanced Configuration and Power Interface | Интерфейс описания платформы |
| **AER** | Advanced Error Reporting | Расширенная обработка ошибок PCIe |
| **ARI** | Alternative Routing-ID Interpretation | Альтернативная нумерация функций |
| **BAR** | Base Address Register | Регистр базового адреса окна |
| **BDF** | Bus/Device/Function | Адрес функции на шине |
| **ECAM** | Enhanced Configuration Access Mechanism | Доступ к конфигурации через окно памяти |
| **FDT** | Flattened Device Tree | Дерево устройств в двоичном виде |
| **IOMMU** | Input/Output Memory Management Unit | Блок трансляции адресов DMA |
| **MCFG** | Memory Mapped Configuration Space | Таблица ACPI с описанием окон ECAM |
| **MMIO** | Memory-Mapped I/O | Доступ к регистрам через адреса памяти |
| **MSI** | Message Signaled Interrupt | Прерывание через запись в память |
| **MSI-X** | MSI Extended | Расширенный вариант MSI |
| **PCIe** | PCI Express | Последовательная шина расширения |
| **RC** | Root Complex | Корневой комплекс |
| **SR-IOV** | Single Root I/O Virtualization | Виртуализация устройств |
| **TLP** | Transaction Layer Packet | Пакет транзакционного уровня |
| **Мост** | Bridge | Функция, соединяющая две шины |
| **Корневой порт** | Root Port | Порт корневого комплекса |
| **Сегмент** | Segment Group | Независимое адресное пространство PCI |

## Обозначения

| Сокращение | Значение |
|------------|----------|
| `RW` | Чтение и запись |
| `RO` | Только чтение |
| `WO` | Только запись |
| `RW1C` | Запись единицы очищает бит |
| `RC` | Чтение очищает |
| `—` | Зарезервировано |
| `x` | Не определено |

## Единицы

| Значение | Смысл |
|----------|-------|
| 4 КБ | Размер configuration space функции |
| 32 КБ | Размер на устройство (8 функций) |
| 1 МБ | Размер на шину (32 устройства) |
| 256 МБ | Максимум на сегмент (256 шин) |

## Формулы

```text
адрес = base + (bus << 20) + (device << 15)
             + (function << 12) + offset

размер BAR = (~(mask & ~0xF)) + 1

```
