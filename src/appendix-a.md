# Приложение A. Карта configuration space

Сводные таблицы смещений. Все адреса — относительно начала
configuration space функции.

## Заголовок типа 0

| Смещение | Размер | Поле | Тип |
|----------|--------|------|-----|
| `0x00` | 2 | Vendor ID | RO |
| `0x02` | 2 | Device ID | RO |
| `0x04` | 2 | Command | RW |
| `0x06` | 2 | Status | RW1C |
| `0x08` | 1 | Revision ID | RO |
| `0x09` | 1 | Programming Interface | RO |
| `0x0A` | 1 | Subclass | RO |
| `0x0B` | 1 | Class Code | RO |
| `0x0C` | 1 | Cache Line Size | RW |
| `0x0D` | 1 | Latency Timer | RW |
| `0x0E` | 1 | Header Type | RO |
| `0x0F` | 1 | BIST | RW |
| `0x10` | 4 | BAR0 | RW |
| `0x14` | 4 | BAR1 | RW |
| `0x18` | 4 | BAR2 | RW |
| `0x1C` | 4 | BAR3 | RW |
| `0x20` | 4 | BAR4 | RW |
| `0x24` | 4 | BAR5 | RW |
| `0x28` | 4 | CardBus CIS Pointer | RO |
| `0x2C` | 2 | Subsystem Vendor ID | RO |
| `0x2E` | 2 | Subsystem ID | RO |
| `0x30` | 4 | Expansion ROM Base | RW |
| `0x34` | 1 | Capabilities Pointer | RO |
| `0x38` | 4 | Reserved | — |
| `0x3C` | 1 | Interrupt Line | RW |
| `0x3D` | 1 | Interrupt Pin | RO |
| `0x3E` | 1 | Min Grant | RO |
| `0x3F` | 1 | Max Latency | RO |

## Заголовок типа 1

| Смещение | Размер | Поле | Тип |
|----------|--------|------|-----|
| `0x00` | 2 | Vendor ID | RO |
| `0x02` | 2 | Device ID | RO |
| `0x04` | 2 | Command | RW |
| `0x06` | 2 | Status | RW1C |
| `0x08` | 1 | Revision ID | RO |
| `0x09` | 1 | Programming Interface | RO |
| `0x0A` | 1 | Subclass | RO |
| `0x0B` | 1 | Class Code | RO |
| `0x0C` | 1 | Cache Line Size | RW |
| `0x0D` | 1 | Latency Timer | RW |
| `0x0E` | 1 | Header Type | RO |
| `0x0F` | 1 | BIST | RW |
| `0x10` | 4 | BAR0 | RW |
| `0x14` | 4 | BAR1 | RW |
| `0x18` | 1 | Primary Bus Number | RW |
| `0x19` | 1 | Secondary Bus Number | RW |
| `0x1A` | 1 | Subordinate Bus Number | RW |
| `0x1B` | 1 | Secondary Latency Timer | RW |
| `0x1C` | 1 | IO Base | RW |
| `0x1D` | 1 | IO Limit | RW |
| `0x1E` | 2 | Secondary Status | RW1C |
| `0x20` | 2 | Memory Base | RW |
| `0x22` | 2 | Memory Limit | RW |
| `0x24` | 2 | Prefetchable Memory Base | RW |
| `0x26` | 2 | Prefetchable Memory Limit | RW |
| `0x28` | 4 | Prefetchable Base Upper 32 | RW |
| `0x2C` | 4 | Prefetchable Limit Upper 32 | RW |
| `0x30` | 2 | IO Base Upper 16 | RW |
| `0x32` | 2 | IO Limit Upper 16 | RW |
| `0x34` | 1 | Capabilities Pointer | RO |
| `0x38` | 4 | Expansion ROM Base | RW |
| `0x3C` | 1 | Interrupt Line | RW |
| `0x3D` | 1 | Interrupt Pin | RO |
| `0x3E` | 2 | Bridge Control | RW |

## Command

| Бит | Имя |
|-----|-----|
| 0 | IO Space |
| 1 | Memory Space |
| 2 | Bus Master |
| 3 | Special Cycles |
| 4 | Memory Write & Invalidate |
| 5 | VGA Palette Snoop |
| 6 | Parity Error Response |
| 7 | Wait Cycle Control |
| 8 | SERR# Enable |
| 9 | Fast Back-to-Back Enable |
| 10 | Interrupt Disable |

## Status

| Бит | Имя |
|-----|-----|
| 3 | Interrupt Status |
| 4 | Capabilities List |
| 5 | 66 MHz Capable |
| 7 | Fast Back-to-Back Capable |
| 8 | Master Data Parity Error |
| 11 | Signaled Target Abort |
| 12 | Received Target Abort |
| 13 | Received Master Abort |
| 14 | Signaled System Error |
| 15 | Detected Parity Error |

## Формат BAR

| Бит | Память | Ввод-вывод |
|-----|--------|------------|
| 0 | 0 | 1 |
| 2:1 | Разрядность | Зарезервировано |
| 3 | Prefetchable | — |
| 31:4 | Адрес | Адрес (с 2) |

## Возможности

| ID | Имя |
|----|-----|
| `0x01` | Power Management |
| `0x04` | Slot Identification |
| `0x05` | MSI |
| `0x0D` | Subsystem Vendor ID |
| `0x10` | PCI Express |
| `0x11` | MSI-X |

## Расширенные возможности

| ID | Имя |
|----|-----|
| `0x0001` | Advanced Error Reporting |
| `0x0002` | Virtual Channel |
| `0x0003` | Device Serial Number |
| `0x0004` | Power Budgeting |
| `0x000D` | Access Control Services |
| `0x000E` | Alternative Routing-ID |
| `0x0010` | SR-IOV |
| `0x0018` | Latency Tolerance Reporting |
| `0x001D` | Downstream Port Containment |

## Коды классов

| Код | Класс |
|-----|-------|
| `0x00` | Не указан |
| `0x01` | Массовое хранение |
| `0x02` | Сеть |
| `0x03` | Дисплей |
| `0x04` | Мультимедиа |
| `0x05` | Память |
| `0x06` | Мост |
| `0x07` | Простой обмен |
| `0x08` | Базовая система |
| `0x09` | Ввод |
| `0x0C` | Последовательная шина |
| `0xFF` | Другое |
