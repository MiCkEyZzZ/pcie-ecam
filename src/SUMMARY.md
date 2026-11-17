# Содержание

[Введение](README.md)

# Предисловие

- [О руководстве](preface.md)
- [Аудитория и соглашения](preface-audience.md)
- [Дополнительная литература](further-reading.md)

# 1. Введение

- [1.1. Что такое host bridge](ch01-about.md)
- [1.2. Место ECAM в топологии PCIe](ch01-ecam-role.md)
- [1.3. Задачи драйвера](ch01-tasks.md)
- [1.4. Границы применимости](ch01-scope.md)

# 2. Функциональное описание

- [2.1. Обзор механизма ECAM](ch02-overview.md)
- [2.2. Адресация bus/device/function](ch02-addressing.md)
- [2.3. Раскладка окна ECAM](ch02-ecam-layout.md)
- [2.4. Таблица MCFG](ch02-mcfg.md)
- [2.5. Сегментные группы](ch02-segments.md)
- [2.6. Configuration space и его типы](ch02-config-space.md)
- [2.7. Заголовки функций и мостов](ch02-headers.md)
- [2.8. Перечисление шины](ch02-enumeration.md)
- [2.9. Распределение ресурсов](ch02-resources.md)
- [2.10. Маршрутизация configuration-запросов](ch02-routing.md)

# 3. Модель программиста

- [3.1. Общие сведения](ch03-about.md)
- [3.2. Заголовок типа 0](ch03-header0.md)
- [3.3. Команда и статус](ch03-command-status.md)
- [3.4. Базовые адреса BAR](ch03-bars.md)
- [3.5. Список возможностей](ch03-capabilities.md)
- [3.6. Заголовок типа 1](ch03-header1.md)
- [3.7. Расширенное пространство](ch03-extended.md)
- [3.8. Правила доступа](ch03-access-rules.md)

# 4. Проектирование драйвера

- [4.1. Обзор архитектуры](ch04-overview.md)
- [4.2. Абстракция доступа к конфигурации](ch04-traits.md)
- [4.3. Вычисление адреса](ch04-address.md)
- [4.4. Чтение и запись](ch04-read-write.md)
- [4.5. Ошибки и их обработка](ch04-errors.md)
- [4.6. Требования к безопасности](ch04-safety.md)
- [4.7. Состояние перечисления](ch04-enum-state.md)
- [4.8. Нумерация шин](ch04-bus-numbers.md)
- [4.9. Размещение BAR](ch04-bar-alloc.md)
- [4.10. Порядок инициализации](ch04-init.md)

# 5. Интеграция с платформой

- [5.1. ACPI и MCFG](ch05-acpi.md)
- [5.2. Описание в дереве устройств](ch05-dt.md)
- [5.3. Особенности реализаций](ch05-quirks.md)
- [5.4. DMA и когерентность](ch05-dma.md)
- [5.5. Порядок запуска на плате](ch05-platform.md)

# 6. Эталонная реализация в Linux

- [6.1. Устройство pci-host-generic](ch06-linux.md)

# 7. Проверка и отладка

- [7.1. Стенды и эмуляторы](ch07-testing.md)
- [7.2. Типичные ошибки](ch07-debug.md)

# Приложения

- [Приложение A. Карта configuration space](appendix-a.md)
- [Приложение Б. Глоссарий](glossary.md)
