# Перевод самописного UI на компоненты дизайн-системы (rt-ui)

Причина: часть интерфейса нарисована руками на Tailwind, хотя в `@lct-testkit/rt-ui` есть готовый компонент. Заказчик хочет **дизайн-систему везде, где она что-то умеет**. Своё (Tailwind) допустимо только для раскладки (flex/grid/отступы), текста и карточек-секций (компонента «Card» в rt-ui нет).

## Что на что менять

| Самописное | Заменить на |
|---|---|
| Плашка-баннер: `<div role="alert|status" class="… bg-warning-soft|bg-danger-soft|bg-info-soft …">` (около 35 мест) | `<Notice tone="info|warning|error|success" title="…" actions={[{label, onclick}]}>текст</Notice>` из `$lib/ui` (обёртка над `InlineNotification`). Кнопка-ссылка внутри баннера («К подписанию») → `actions` |
| Пилюля статуса `<span class="rounded-full …">` | `<StatusChip label tone size>` из `$lib/ui` (внутри rt-ui `Badge`). Счётчик-кружок → rt-ui `Counter` |
| Ряд переключаемых пилюль-фильтров | `<FilterChips>` из `$lib/ui` (rt-ui `Chip`) |
| Голая `<button class="…">` с текстом/иконкой | `Btn` / `IconBtn` из `$lib/ui` (rt-ui `Button`/`IconButton`); выбираемая пилюля → rt-ui `Chip`; вкладки → rt-ui `TabsGroup`/`TabsItem` |
| Голый `<input>` / `<textarea>` / `<select>` | rt-ui `Input` / `InputNumberStepper` / `TextArea` / `Select` / `Checkbox` / `Switch` / `RadioGroup` (стиль как в соседних формах) |
| Самописный спиннер / «Загрузка…» | rt-ui `Loader` (или `Skeleton` из `$lib/ui` для списков) |
| Самописный тултип на CSS | rt-ui `Tooltip` |

Оставить как есть можно только то, для чего в rt-ui нет аналога: кликабельная карточка/строка целиком (`<a>`/`<button>` вокруг блока), ссылка в тексте, разметка. Для этого случая — объяснить в одной строке комментария, почему.

## Правила
* Только Tailwind для раскладки, никакого `<style>`. Радиусы: `rounded-sm|md|lg|full` (**не** `rounded-s|m|l`, тест `class-names.test.ts` упадёт).
* Логику и тексты не менять, только компонент. Поведение (клавиатура, фокус, aria) не должно ухудшиться.
* После каждой пачки правок: `pnpm check` (0 ошибок), `pnpm test`, затем посмотреть экран: `node tools/shot.mjs --as <роль> --url <путь> --sizes desktop,phone,phone-s --name <имя>` и **прочитать PNG** (проверить вид, отсутствие горизонтальной прокрутки).
* Сценарии `tools/scenarios/*.mjs` в твоей области должны продолжать проходить (если ищут элемент по старой разметке, поправь селектор в сценарии).
* Работать только в своих папках. Коммитов не делать. Sub-агентов не создавать.
