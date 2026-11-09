# Семантика сервера JustMC: что известно

Факты, которые не видно из сигнатур. Помечено, откуда известно.
Открытые вопросы проверяет `bench/server-check.jc`: загрузить в мир и прислать вывод чата.

## Типы переменных (от владельца проекта, 2026-11-02)

| Тип | Видна | Живёт | В JustCode |
|---|---|---|---|
| сохранённая (SAVE) | во всём коде | переживает перезапуск мира | `save var` |
| игровая (GAME) | во всём коде | до перезапуска мира | `game var` |
| локальная (LOCAL) | в своём «потоке»; **передаётся в вызванные функции** | пока вызванный код полностью не завершится | `var` |
| строчная (LINE) | только в своей строке, никуда не передаётся | пока строка не закончит выполнение | `line var` (JSON `"scope": "line"`, текст `%var_line(x)`; проверено bench/line-check.jc) |

## Жизнь «потоков» (владелец, 2026-11-02)

Поток, привязанный к сущности (событие игрока, процесс, запущенный с целью), **обрывается на ходу**,
когда цель исчезает (игрок вышел, моб умер). `repeat::forever` в `player_join` умирает с выходом игрока.
Вызов `proc()` из JustCode компилируется в `start_process(target_mode=CURRENT_TARGET, local_variables_mode=COPY)` —
тоже привязан к игроку. Для цикла на весь мир: запускать из `world_start` или
`code::start_process("proc", target_mode = "NO_TARGET")`.

Проверено server-check-22 (2026-11-04):
- `local_variables_mode = "COPY"` — процесс получает копию `var` вызывающего; `"SHARE"` — общие (запись в процессе
  видна вызывающему); `DONT_COPY` (по умолчанию у `start_process`) — пусто (0).
- `/build` (и деплой) убивает идущие процессы СРАЗУ: процесс с `wait` в цикле не сделал ни шага после остановки.
- `event<world_stop>` выполняется синхронно в момент остановки: 20 000 шагов цикла (≈8 мс) прошли, запись в save var
  сохранилась; код после `code::wait(...)` в нём НЕ выполняется. Это последний шанс: вернуть ресурсы, записать
  save var — без ожиданий.

Проверено server-check-26 / 26b (2026-11-04):
- `/build` (и деплой): сначала `player_quit` каждого игрока (игрок ещё в `value::players`), потом в ТОТ ЖЕ тик
  `world_stop` — игроков уже 0: `select::all_players` + процесс FOR_EACH_IN_SELECTION в нём никого не находит,
  функция из него идёт, `%player%` в ней не подставляется. Пер-игровое сохранение — в `player_quit`, мировое — в `world_stop`.
- Процесс NO_TARGET, запущенный из события игрока: `%uuid%` и `%player%` остаются ТЕКСТОМ (не подставляются),
  `value::uuid` = 0. Процесс FOR_EACH_IN_SELECTION из NO_TARGET-процесса работает за каждого игрока (`%player%` — его).
- `log()` (сообщение всем) в `world_start` теряется: игрока в мире ещё нет. Печать при старте — в `player_join`.
- Пакет custom click (мод → сервер, 1.21.6+) ловит `event<player_custom_click>` без чата и окон:
  `value::event_custom_click_key` — id пакета (`jmvcc:rpc`), `value::event_custom_click_dialog_view_data` —
  СЛОВАРЬ из SNBT-данных (`{a:1,b:"x",c:[1,2]}` → `{c=[1, 2], b=x, a=1}`, `to_json` — `{"c":[1,2],"b":"x","a":1}`);
  данные-строка (не compound) приходят пустым словарём (server-check-25).
- Сообщение `#jmvcc ...` с `world::cancel_event()` в `player_chat` в чате мира не видно (server-check-24).

## Проверено

| Что | Ответ | Источник |
|---|---|---|
| База индексов (списки) | с **0** | владелец, 2026-11-02 |
| `code::controller_exception(e) { ... }` (основа try/catch) | без ошибки `e` **не трогается** (остаётся прежнее значение — сбрасывать перед блоком); при ошибке сервера тело обрывается на ней, в `e` — id ошибки (`expected_but_got`), код ПОСЛЕ блока идёт; ловит ошибку и во вложенной функции, и после `code::wait` в теле; `code::call_exception(id, msg, ERROR)` в теле → в `e` — **сообщение** (`msg`), не id. Без ловли: `call_exception(..., WARNING)` — предупреждение в лог, код идёт дальше; `ERROR` — строка обрывается | server-check-28 (2026-11-04) |
| `code::controller_exception`: вложенный, выходы, типы ошибок | вложенный блок ловит свою ошибку, внешний не срабатывает (e внешнего не тронута); ошибка / `call_exception(ERROR)` в коде после внутреннего блока — во внешний (e = сообщение). `return` (и `return 5`) в функции из тела — выход из функции, вызывающий идёт дальше; `stop_repeat` / `skip_iteration` из тела — выход из цикла / следующий шаг (тело не считается ошибкой, e не тронута). **`control_end_thread` в теле выходит только из блока**: код после блока идёт (и в процессе, и в цикле — следующий шаг, и из функции, вызванной в теле, и из вложенного блока — только из него). `call_exception(..., WARNING)` в теле — предупреждение в лог, тело идёт дальше, e не тронута; `FATAL` — «Критическая ошибка» в лог, тело тоже шло дальше, e не тронута; пустой id — работает, e = сообщение. Непойманный ERROR в логе: «Вызвать ошибку: <сообщение> @ …» — id не виден нигде. В e всегда **текст** (`is_type TEXT`, `to_json` — `"expected_but_got"`). `number_in_range(текст, …)` и `текст > 0` (не число) — **ошибка сервера** expected_but_got | server-check-34 (2026-11-04) |
| line var / local, записанные в теле controller_exception | видны после блока (и когда тело потом оборвалось ошибкой); line var функции — своя у каждого вызова (рекурсия: `line var mine = n` → 2, 1, 0), а параметр-local общий | server-check-35 |
| Цена проверок (нетто, `a = i` = 1, K = 1000 за тик, 2 круга) | `if (!variable::is_type(z, "NUMBER"))` ≈7.3, `z != 0` ≈5.7, `!number_in_range(z, 0, 0)` ≈1.1, `z > 0` ≈0.8, пустой `controller_exception(q) { }` ≈0.5 | server-check-34 (2026-11-04) |
| Сахар `try / catch / throw / match` (CodeVisitor) | как задумано и без `-O`, и с `-O`: catch с id ошибки, throw → e = сообщение, вложенный (только внутренний), rethrow во внешний, `return` в процессе внутри try обрывает поток, break / continue цикла, `return 5` из функции сквозь try, try в цикле — заново каждый шаг, рекурсия — catch только у уровня с ошибкой; match по числам (1, 2 / 3 / else), текстам, enum (голое имя члена) | server-check-36 |
| Нет элемента: `get_list_value` за концом / с -1, `get_map_value(_single)` без ключа, без `default_value` | **null**: в тексте `0`, `to_json` — `null`, `%math` и `< >` читают 0, `+= 0` делает числом 0, `variable::exists` — да, но `== 0`, `== "0"`, `== ""` и `number_in_range(0, 0)` — **ложь**. С `default_value = 7` — 7. Сахар `l[i]` / `m[k]` передаёт `default_value = 0` | server-check-26c, -26d |
| Текст из `split_text` (`"0"`) и число | `"0" == 0` — ложь, `"0" < 1` — истина (сравнения < > читают текст числом) | server-check-26 (`#jmvcc build 0` → 90, 2026-11-04) |
| Переменная, которой не присваивали, в тексте | `0` | server-check №1 |
| Строчная переменная и `code::wait` | **переживает** ожидание внутри своей строки | bench/line-check.jc №4 |
| `return` в function (`control_return_function`) | выходит только из функции, код после вызова идёт дальше; значение через общую локальную `jmcc.return` доходит | bench/ops-check.jc, 2026-11-02 |
| `&&`, `||`, `!`, `else` с флагом, сравнение текста `==`, `repeat::while(i < 3)` | работают как задумано (10/10) | bench/ops-check.jc |
| Пустой словарь | `variable::create_map_from_values([], [])` → размер 0, текст `{}` | server-check-2 №3a |
| `set_map_value` на переменной, которой не присваивали | **ошибка сервера**: «Ожидал аргумент "Словарь для изменения" с типом Словарь, но получил Число» — **строка кода обрывается**. Сначала `m ??= variable::create_map_from_values([], [])` | server-check-2 №3b |
| Клик в меню (`player_click_inventory`) | `event_slot` с **0**; `event_item` — кликнутый предмет (NBT-компоненты); `open_inventory_custom_id` — custom_id из show_menu; `event_slot_type` = container | server-check-2 №1 |
| Ошибка сервера в чате | JSON-компонент: `❌ Ошибка`, «Действие <имя> остановило строку кода из-за ошибки: <причина>», click_event `/editor tp x y z` — координаты блока в мире кода; цвет #FF6E6E | лог клиента 2026-11-02 |
| `give_items(item("arrow", count = 16), amount = 2)` | выдаёт **2** — amount заменяет count, не умножает | server-check-2 №2 (мост) |
| Порядок словаря после `sort_any_map` | сохраняется: for_each и get_map_key_by_index идут в отсортированном порядке | server-check-2 №3c |
| `\n` в тексте и список с `merging = SEPARATE_LINES` | переводы строк работают | server-check-2 №5b, 5c |
| Компонент с hover/click (`set_component_hover`/`set_component_click`) | работает: show_text и copy_to_clipboard | server-check-2 №5a |
| Цвета в тексте | `&a` работает (цвет до конца текста); MiniMessage `<red>` и `§b` — НЕ работают, видны как текст | server-check-2 №5d |
| `code::stop_repeat()` в функции, вызванной из `repeat::forever` | **останавливает** цикл вызывающего | server-check-2 №6a |
| `break` вне цикла в function (= `control_end_thread`), вызванной из цикла события | **обрывает всю строку события**: ни цикл, ни код после него не выполняются. Компилятор предупреждает | bench/sneak/sneak-check.jc (мост) |
| `event<player_move>` | срабатывает **каждый тик**, пока игрок движется (20 в секунду, и при падении); стоя на месте — 0. Поворот камеры не проверен | server-check-2 №4 (мост, клавиши) |
| `value::event_block_type` | полное состояние блока: `"minecraft:light_weighted_pressure_plate[power=0]"`. **`==` с id (и с `minecraft:id`) всегда ложно** — тип блока проверять `world::event_block_equals([item("id")])` или `world::block_equals(value::event_block, [item("id")])` (оба работают). Компилятор предупреждает | server-check-3 №3g (мост) |
| `value::event_block` | местоположение блока (`{"x":0.5,"y":4.5,"z":0.5}`), не id | server-check-3 №3e |
| Стоять на нажимной плите | `player_interact` с `event_interaction == "physical"` срабатывает ~2 раза в секунду, пока стоишь (6 за 3 с) | server-check-3 №3e |
| `player::set_boss_bar(..., progress)` | шкала **0..100** (50 = половина; 0.5 — почти пусто) | server-check-3 №3c (скрин) |
| `world::remove_scoreboard` несуществующего табло | ничего не происходит, строка кода идёт дальше | server-check-3 №3d |
| `player_right_click` | **один раз** на клик (hand = `"hand"`, interaction = `"right_click_air"`) при пустой второй руке. ПКМ **пустой рукой в воздух** — события нет (клиент ничего не шлёт) | server-check-3 №3a |
| Клик по своему инвентарю, пока открыто меню `show_menu` | идёт в **`player_click_own_inventory`**, не в `player_click_inventory`; `event_slot` — номер в своём инвентаре (хотбар 0..8, основной 9..35), `event_slot_type` = `"container"`. В `player_click_inventory` попадает только верхняя часть (меню) | server-check-3 №3b |
| `value::players` / `value::all_players_names` в мировом коде (верхний уровень, без цели) | работают: `1` и `[Morphill]` (список имён) | server-check-4 №4a |
| `event<player_join>` и `/play` | приходит **и тем, кто уже в мире** при запуске игры (`/play`, `jmvcc client deploy`) — инициализация в player_join сработает | server-check-4 №4b |
| `==` двух локаций | работает: одинаковые `value::location`, литералы `location(1, 2, 3)` и два `value::event_block` одной плиты равны | server-check-4 №4c, bench/sneak/block-eq.jc |
| `get_map_value` нет такого ключа, без `default_value` | **0** | server-check-4 №4d |
| `value::open_inventory_custom_id` в `player_click_own_inventory` | есть (id открытого меню); `event_inventory_click_type` = `"shift_left"` для shift-клика; отменённый shift-клик **не** переносит предмет в меню | server-check-4 №4e |
| Табло-сайдбар: `world::create_scoreboard` + `set_scoreboard_score(id, "Строка", n)` + `player::show_scoreboard(id)` | показывается справа; `text` — подпись строки, `&`-цвета в названии работают | server-check-4 №4f (скрин) |
| `give_items(item("golden_apple", count = 3))` без `amount` | выдаёт **3** (count предмета); `amount` — заменяет | server-check-4 №4g, server-check-2 №2 |
| ПКМ с книгой в руке | `player_right_click` срабатывает; `value::main_hand_item` — эта книга | server-check-4 №4h |
| `player_death` от урона не от игрока (`player::damage`) | `%player%` = умерший; `%killer%` и `%victim%` **не подставляются** — остаются текстом `%killer%` | server-check-3 №3f |
| Порядок при `/play` | сначала мировой код (верхний уровень), потом `player_join`; join видит `game var`, заданные мировым кодом | server-check-5 №5o |
| `world::spawn_mob(..., health = 200, "&4&lKing")` → `value::uuid<last_entity>` | uuid этого моба; здоровье 200 работает (без set_max_health); имя возвращается с §-кодами (`§4§lKing`) | server-check-5 №5b |
| `world::has_entity([uuid])` | живой моб — да; `""` — нет | server-check-5 №5b |
| Блоки и сущности при `/play` и `/build` | всё, что код поставил (`set_block`, `set_region`…) и создал (мобы, стойки, дисплеи, НПС), **остаётся** в мире — `/play` и `/build` его не сбрасывают. Код в мировом коде (world_start) выполняется при КАЖДОМ запуске: статичный НПС без проверки будет плодиться | владелец, server-check-5 |
| Мобы и `/play` (перезапуск мира) | мобы **остаются** в мире, а `game var` с их uuid обнуляются. Убрать всех: `select::all_mobs()` + `entity::remove()` в мировом коде | server-check-5, bench/sneak/cleanup.jc |
| Выбор `select::...` в процессе `NO_TARGET` | действия без селектора (`player::message`, `entity::damage`) идут **выбранному**; `value::x<current>` **читает выбранного** (`health<current>` = 200, `uuid<current>` = uuid моба); `value::x` **без** селектора выбранного не видит (0) | server-check-5 №5a, №5a2 |
| Прочитать `value::` выбранной сущности | проще всего `value::x<current>` (см. выше); или `code::start_process("p", target_mode = "FOR_EACH_IN_SELECTION")` — в процессе `value::health`, `value::uuid` — этой сущности. `CURRENT_SELECTION` и событие игрока после `select` — `value::` по-прежнему про игрока | bench/sneak/sel-read.jc |
| После `select::reset()` в `NO_TARGET` | действие без селектора (`player::message`) не уходит никому, строка идёт дальше | server-check-5 №5a |
| `select::entity_by_name(["нет такого"])` + `entity::remove()` | ничего не удаляет, ошибки нет | server-check-5 №5a |
| `entity::damage(5, "Morphill")` выбранному мобу | засчитывается как удар игрока: `player_damage_entity` с `%player%` = Morphill, `event_damage_cause` = `entity_attack`, `event_final_damage` = 4.92 (с бронёй); без источника — cause `custom` | server-check-5 №5d, №5e |
| `value::health` в `entity_take_damage` | здоровье **до** удара (`event_final_damage` — сколько снимет) | server-check-5 №5d |
| `value::uuid` в `entity_take_damage` / `entity_death` | uuid этой сущности (раненой / умершей) | bench/sneak/boss-test.jc |
| Рецепт босса docs/recipes-boss.jc | прогнан вживую: призыв звездой, боссбар, таблица урона, финал «Top damage», смерть по uuid | bench/sneak/boss-test.jc |
| Зомби днём | горит: `entity_combust` с `event_combust_reason` = `unknown`, урон `fire_tick` по 1 в секунду | server-check-5 №5g |
| Поток `entity_death` | **живёт** после `code::wait` (сущность уже мертва, код идёт дальше) | server-check-5 №5f |
| `var b = a` для списка | **копия**: `b[0] = "z"` не меняет `a` | server-check-5 №5l |
| `cos(120)`, `sin(90)` | градусы: `-0.5000000000` (10 знаков в тексте) и `1` | server-check-5 №5t |
| `player::set_slot_item(item, 8)` | 8 — последний слот хотбара (слоты с 0) | server-check-5 №5s (скрин) |
| `&`-цвета в названии боссбара | работают (`&4&l` — тёмно-красный жирный) | server-check-5 №5c (скрин) |
| `player::display_hologram(loc, "A\n&cB")` | две строки, `&`-цвета работают; вторая голограмма **в той же точке заменяет** первую, в другой точке — отдельная | server-check-6 №6h (скрины) |
| Состояние блока в действиях с блоками | текстом в аргумент блока: `world::set_block("oak_stairs[facing=north,half=top]", loc)`, `"grass_block[snowy=true]"` — свойства как в MC | владелец; server-check-7 №7b (скрин: трава в снегу, ступенька вверх ногами) |
| `spawn_mob` → `value::uuid<last_entity>` в мировом коде | работает (не только в событии игрока) | server-check-7 №7n |
| «НПС один раз»: `save var npc` + `npc ??= ""` + `if (!world::has_entity([npc])) { spawn; npc = value::uuid<last_entity> }` | после повторного /play НПС в мире ОДИН | server-check-7 №7n, bench/sneak/count-mobs.jc |
| Сообщение из мирового кода в самом начале запуска | при повторном /play может не дойти до игрока (не успел войти в мир); после `code::wait(2, time_unit = "seconds")` доходит | server-check-7 №7n (наблюдалось 1 раз) |
| Цвета имени предмета: `item("stick", "&#ff8800Оранж &6&lЖирное &eНЕжирное")` | hex ровно #ff8800, жирность сбрасывается цветовым кодом (новый конвертер LegacyText) | server-check-7 №7c (пиксели скрина) |
| `repeat::on_range(1, n)`, `n` меняется в теле | конец читается **один раз** (3 шага при n=3→10) | server-check-8 №8a1 |
| `for_each_in_list(l)`, в теле `l.list_append_value(...)` | цикл **обрывается** после первого шага (1 шаг вместо 2–3), без ошибки — не меняй список, по которому идёшь | server-check-8 №8a2 |
| `code::stop_repeat()` / `code::skip_iteration()` | срабатывают **сразу**: код после них в той же итерации не выполняется | server-check-8 №8a3, №8a4 |
| Функция с `code::wait` внутри | вызывающий **ждёт** её конца (после вызова видно, что она записала после wait) | server-check-8 №8a5 |
| Ошибка сервера внутри вызванной функции | обрывает и **строку вызывающего**: код после вызова не выполняется | server-check-8 №8a6 |
| `variable::equals(x, [5, 6], compare_type = "ANY")` | x=6 — да, x=7 — нет; `not_equals("a", ["b","c"])` — да, `("b", ["b","c"])` — нет (не равно ВСЕМ) | server-check-8 №8b1, №8b2 |
| `variable::number_in_range` | NOT_INCLUDE — без краёв, INCLUDE_FIRST — [min, max), INCLUDE_ALL — [min, max] | server-check-8 №8b4 |
| `variable::multiple_equals([1,2], [1,2], ALL)` | да; `[1,2]` против `[2,1]` — нет (сравнение по парам) | server-check-8 №8b5 |
| `variable::list_value_equals(gl, 1, ["b"])` | индексы с **0**; индекс вне списка — ложь, без ошибки | server-check-8 №8b7 |
| `variable::list_is_empty([l])` | пустой список — да, непустой — нет | server-check-8 №8b8 |
| `%val(id)` в тексте | `%val(timestamp)`, `%val(x_coordinate)` = 0.5, `%val(current_health)` = 20 (а `%val(health)` = 0 — такого id нет), неизвестный id → 0 | server-check-8 №8c3 |
| `%length(gl)` / `%length_local(ll)` | длина списка и словаря; несуществующая переменная → **-1** | server-check-8 №8c7 |
| `%math(%length(gl)-1)`, `%index(gl, 0)`, `%entry(gm, k)` | работают: 2; `a` (индекс с 0); 7; нет ключа → 0 | server-check-8 №8c8–8c10 |
| `%math(floor(75000/60000))`, `%math(10/4)` | 1 и 2.5 (дробь с точкой) | server-check-8 №8c12 |
| `%%` в тексте | **не экранирование**: `%%val` осталось как есть, а `%%math(...)` съело часть текста — не пиши `%%` | server-check-8 |
| `variable::format_timestamp(ms, "mm:ss", "UTC")` | 75000 → `01:15`; 3725000 → `02:05` (часы теряются), `"H:mm:ss"` → `1:02:05`; `"mm:ss.SSS"` → `01:15.123`; `"m:ss.S"` → `1:15.123` (S не обрезает) | server-check-8 №8d |
| `var y = x` из неприсвоенной `x` (local и game) | `y` = **0** и `variable::exists(y)` — да (сама `x` по-прежнему не существует) | server-check-9 №9a |
| Событие, которое вызвало действие (`player::damage` → `player_take_damage`) | выполняется **синхронно**, до возврата из действия: сразу после `player::damage` запись обработчика уже видна | server-check-9 №9b |
| `code::controller_async_run() { ... }` | **не работает**: ошибка «Action should not be executed directly», строка обрывается — в событии, функции и процессе (компилятор предупреждает `async-run-broken`) | server-check-9 №9c |
| Две функции с одним именем (в одном файле) | вызов выполняет **обе** по порядку объявления (компилятор предупреждает `duplicate-function`) | server-check-9 №9d |
| `%val(id)` без селектора | берёт цель по умолчанию, как `value::x`: в `entity_take_damage` — повреждённую сущность (здоровье до удара), в процессе без цели после `select::all_players()` — выбранного (как `value::x<current>`) | server-check-10 №10a-c |
| `gl[9]` (get_list_value вне списка) | **0**, переменная создаётся (`exists` — да), ошибки нет. Значит `l[i] == 0` вне списка — истина, а `list_value_equals(l, i, [0])` — ложь | server-check-10 №10d |
| `variable::list_is_empty([x])` | неприсвоенная — **да**; текст `"abc"` — нет; пустой словарь — да | server-check-10 №10e |
| `if (x == 0)` / `x != 0` / `multiple_equals([x],[0])` / `equals(0, [x])` с неприсвоенной `x` | неприсвоенная — **0** везде: == 0 да, != 0 нет, оба списка — да | server-check-11 №11a |
| `not if` условия по сущностям с двумя выбранными (одна подходит / обе) и с нулём выбранных | `if` с двумя целями — **да**, если подходит хоть одна (и обе); `not if` — всегда точное дополнение `if`; ноль целей: `if` нет, `not if` да | server-check-11 №11b |
| `entity::set_custom_name(value::health)` по 2 выбранным свиньям в процессе (вызван из события игрока) | аргумент вычисляется **один раз** от цели потока (здоровье игрока 20 у обеих), не у каждой цели; `line var h = value::health` там же — тоже 20 | server-check-11 №11c |
| `number_in_range` INCLUDE_LAST; min > max; неприсвоенная | (1,5]: 5 да, 1 нет; `3 in [5,1]` — **нет** (границы не меняются местами); неприсвоенная в [0,5] — да (0) | server-check-11 №11d |
| Текст в `number_in_range` и в `>=` | **ошибка** в обоих («Ожидал аргумент … Число, но получил Текст»), строка обрывается (числовой текст "5" читается числом — без ошибки, server-check-22) | server-check-11 №11e |
| `%val(id)` против `value::x` (в событии игрока, в тексте и в `%math`) | одинаково: `x_coordinate` 0.5, `y_coordinate` 4, `z_coordinate`, `current_health` 19, `timestamp`, `event_final_damage` 1 (в player_take_damage); формат чисел тот же; `%math(%val(current_health)+1)` = 20. Неизвестный id — **0** без ошибки | server-check-12 №12a-b |
| `%length(x)` | список — размер (3); неприсвоенная — **-1**; текст — **длина текста** (3); словарь — размер (1) | server-check-12 №12c |
| `%index(l, 9)` вне списка / `%index(неприсвоенная, 0)`; `%entry(m, нет)` | **0** / **0**; **0** (как `get_map_value` без ключа: 0, переменная создаётся) | server-check-12 №12c-d |
| `get_list_length` / `l[0]` не на списке (неприсвоенная, текст, словарь) | **ошибка** «Ожидал аргумент "Список" с типом Список», строка обрывается | server-check-12 №12d |
| `list_value_equals(l, i, [v])` на краях | индекс -1 — **нет** (а `l[-1]` = 0); 1.5 — как 1 (и `l[1.5]` = элемент 1); не на списке — ошибка, как у `l[i]` | server-check-12 №12e |
| Повторный `/play` в режиме игры; `/build` → `/play` | повторный `/play`: `player_join` срабатывает снова, **старый цикл игрока продолжает работать** (два цикла), `world_start` не срабатывает (мировой цикл один); `/build` и деплой останавливают все циклы. Защита — поколение `%player%_loop = value::timestamp` + `if (%player%_loop != my) { stop_repeat }` (проверено: старый цикл останавливается) | server-check-13 |
| `%val` против `value::x` на длинных дробях | одинаково: x 0.123456789, z 0.333333333, здоровье 19.700000762939453; `%math` печатает с 8 знаками (`%math(%val(current_health)*1)` = 19.70000076); от %val и от переменной с тем же значением — одинаково (на целом 20) | server-check-14 №14a, №14e |
| `list_value_equals` на числах | `[1,2,3]`, индекс 1: с `[2]` — да, с `["2"]` — нет; `t = l[1]; t == "2"` — тоже нет (число ≠ текст в обоих) | server-check-14 №14b |
| `%index_local`, `%index_line`, `%entry_local`, `%entry_line`, `%length_line` | работают как игровые формы (b, v, 2 / y, w, 2); `%index` числа — `2`, в `%math` — работает (+1 = 3); дроби в `%index` — как у переменной (2.5, 0.1, 19.700000762939453) | server-check-14 №14c, №14e |
| Вложенный список-литерал `a = [[1, 2], ["x"]]` | до 2026-11-03 компилятор клал во `values` переменную-список, сервер — одним элементом: длина 1, `a[0] = [[1,2],[x]]`; теперь наружный список — литерал из переменных-вложенных, длина 2 | server-check-15b |
| `/env var list <game\|save> [поиск]` | 9 строк на страницу, сортировка по имени, поиск — подстрока; значение — в hover строки (`{type: number, '7.0&Double'}`, длинное обрезано `... (+N)`); стрелки — `/env var list page N <область> [поиск]`; line/local не видны | server-check-15 |
| `/env var clear <game\|save> [имя]` | имя точное (префикс — «не найдена»), без имени — вся область; `game var x = 0` после чистки не перезапускается до `/play` | server-check-15 |
| Отрицательное число после оператора в `%math` | `5--3` = 8, `5+-3` = 2, `5*-2` = -10, `5/-2` = -2.5, `5-(-3)` = 8, `-5+1` = -4: компилятор пишет `x - -3` как `%math(x--3)` — это верно (не баг) | server-check-16 |
| Текст на верхушке блока | `world::spawn_text_display(location(x+0.5, верх+0.01, z+0.5+0.125*s, 0, -90), ...)` + `entity::set_display_scale<last_entity>(vector(s, s, s), "SET")` + `set_text_display_background<last_entity>("#000000", 0)`: лежит плашмя, читается ровно стоя с юга (лицом на север; yaw 180 — вверх ногами/боком); привязка — нижний край строки, сдвиг +0.125·s по z центрирует; hex-цвет `&#RRGGBB` и символы ⚑ ☠ работают; дисплеи переживают /play — хранить uuid и удалять (`select::entity_by_name([u])` + `entity::remove()`) | server-check-17 |
| `location(...)`, `vector(...)` | только числа-константы; с переменными — `variable::set_all_coordinates(x, y, z, yaw, pitch)` | server-check-17b |
| `player::set_attribute(16, "PLAYER_BLOCK_INTERACTION_RANGE")` | клик по блоку достаёт ~14 блоков (с вышки сапёра до дальнего угла поля) | вживую, сапёр 2026-11-03 |
| Разделитель в чате | `&8&m` + пробелы — сплошная линия; `━` `─` `▬` — со швами; hex `&#RRGGBB` в чате и title работает | server-check-18 (скрин) |
| `event<player_input_event>` | срабатывает на каждое изменение нажатых клавиш; `value::event_input_list` — текст-список: forward, backwards, left, right, jump, sneak (пусто при отпускании); `world::event_has_input("FORWARD"|"BACKWARDS"|"LEFT"|"RIGHT"|"JUMP")` — по одной | server-check-19 |
| Заморозка ходьбы | `player::set_attribute(0, "GENERIC_MOVEMENT_SPEED")` + `(0, "GENERIC_JUMP_STRENGTH")`: игрок не сдвигается (x,z те же), клавиши всё равно приходят в player_input_event | server-check-19 |
| Камера сверху | text_display в `location(x, y, z, 180, 90)` + `player::set_gamemode("SPECTATOR")` + через 5 тиков `player::spectate_target(uuid)`: вид строго вниз, повернуть камеру нельзя; WASD всё равно приходит в player_input_event; Shift выходит из наблюдения (`player_stop_spectating_entity`), возврат — `spectate_target` через 2 тика | server-check-19b (скрин) |
| `player::hide_entity([uuid, ...])` / `hide = false` | прячет серверный дисплей от этого игрока и показывает обратно (в камере змейки табло пропали только у играющего) | вживую, змейка-WASD 2026-11-03 |
| `jmvcc client items` → `item_raw("…")` → `player::give_items` | выданные головы (/hdb, свои текстуры) побайтно совпали с исходными raw, текстуры на месте (скрин хотбара) | вживую 2026-11-03 |
| Shift в камере наблюдателя | одно нажатие иногда приходит в player_input_event дважды в ту же миллисекунду — «двойной Shift» отсекать паузой ~0.7 с; пока Shift зажат, сервер не даёт снова прицепить камеру | вживую (лог), змейка-WASD |
| `/play` посреди игры | player_join срабатывает снова — игрока можно вернуть из SPECTATOR там | вживую, змейка-WASD |
| Хостинг ресурс-паков | `https://raw.githubusercontent.com/hillmorp-ops/rps_Test/main/<путь>.zip` (публичное репо владельца) — сервер принимает (новому репо CDN raw отдаёт 404 первые минуты) | 2026-11-03 |
| `world::set_block` из мирового цикла без цели | без `update_blocks = true` клиент изменений не видел (сервер ставил — проверено логом) | вживую, змейка 2026-11-03 |
| `set_text_display_background(hex, 110)` | ошибка, строка кода обрывается; 55 — работает | вживую, змейка 2026-11-03 |
| `entity::set_text_display_text` | меняет текст дисплея на месте | вживую, змейка 2026-11-03 |
| `player_change_slot` | `value::event_hotbar_slot` — новый слот (0..8) сразу, `value::slot` — ещё старый | вживую (лог), змейка 2026-11-03 |
| `player::set_spawn_point(loc)` | `value::spawn_location` после него не изменился — не работает или не то | вживую, змейка 2026-11-03 |
| Частицы на грани клика: `get_vector_from_block_face(event_block_face)` + `align_location(event_block, KEEP, ALL, BLOCK_CENTER)` + `shift_location_on_vector(…, 0.6)` | частицы появляются на той грани, по которой кликнули (видно от первого лица) | вживую, кликер 2026-11-03 |
| `list_append_value` / `remove_list_value` на неприсвоенной | append — **без ошибки, но переменная остаётся неприсвоенной** (`%length` = -1); remove — **ошибка** «Ожидал Список, получил Число» | server-check-14 №14d |
| Цена действий (bench/cost-1.jc, прогретый сервер, на одно повторение) | присваивание ≈0.45 мкс; вызов пустой функции ≈0.5 мкс; format_timestamp ≈1.3 мкс; ручной mm:ss функцией ≈5.1 мкс; каждое проверенное условие в цепочке else-if ≈2 мкс; текст с %math(%length) ≈0.75 мкс | bench/cost-1.jc |
| Цена условий (bench/cost-2.jc, мкс на повторение) | одно `if (k == 6)` ≈7 (≈ в 10 раз дороже присваивания); 6 плоских if ≈38; цепочка 6 else-if ≈34; одно `variable::equals(k, [1..6], compare_type = "ANY")` ≈7 | bench/cost-2.jc |
| Цена по `value::cpu_usage` под нагрузкой (присваивание = 1) | условие ≈5; equals ANY ≈ одно условие (6 if ≈27); `l[i]` ≈1; текст с %index/%math ≈2; select по uuid + reset ≈4; set_block ≈18–34, с `update_blocks = true` ≈4; мир в покое 9–12%, 1% ≈ 85 присваиваний за тик; подробно dev/research/cpu-bench.md | bench/cpu-0..2.jc |
| Смена карты (16×16×6) | похожие карты: разница списками с update_blocks=true ≈0.15–0.4 мс; копия зоны ≈0.9–1.1 мс почти без разницы, что внутри; set_region/clear ≈0.2 мс за вызов; проверка клетки дороже установки; список точек в set_block — сотни за раз | bench/map-1..5 (dev/research/map-build-bench.md) |
| Лимит изменений мира за тик | ~20 вызовов set_region/clear_region/clone_region/set_block(false) за тик — дальше поток ждёт тик; set_block(update_blocks=true) не ограничен; 40 clone_region за тик — 13 с простоя, CPU 494%, критическая ошибка high_cpu | bench/map-5 |
| update_blocks true против false на настоящей смене (640 блоков/тик, камень↔земля, cpu-3, 2 круга) | cpu: покой 8%, true ≈24%, false ≈48% — true в 2.5 раза дешевле; TPS 20 и 200 тиков за 10 с в обоих — скрытой отложенной цены у true не видно (при запасе сервера) | bench/cpu-3.jc |
| Длина строки кода в модуле (`/module load-url`) | лимита 43 блока **нет**: плоская строка события из 152 блоков (150 `+= 1` и 2 log) и из 1002 блоков (1000 `+= 1`) принята и выполнена целиком (n = 150, n = 1000). С тех пор компилятор строки не режет; `--max-line 43` — старая разбивка на `jmcc.fN` (лишний call_function и копии line var). Длинные строки функций и процессов тоже: функция из 95 и процесс из 93 блоков с вложенными if/else/repeat, ранним return и line var после 43-го блока выполнились верно (логи совпали с interp.ts) | bench/server-check-20.jc, 20b, 21, 2026-11-03 |
| Длина строки кода, новое | функция из 279 блоков: «Не хватает места для загрузки импортируемого модуля»; она же с `--max-line 43` принята и работает. Порог не мерился; запись выше про 1002 блока могла устареть после обновления сервера | 2026-11-09 |
| `text_matches(regular_expressions = true)` | флаг не действует: на `0.5` из 12 шаблонов (`.*`, `[0-9]+`, `0.*`…) совпал только буквальный `0.5`. Рабочая проверка — `t.regex_replace("[0-9.]", "") == ""`, `regex_replace` регэкспы понимает (server-check-38) | 2026-11-09 |
| clone_region с воздухом (ignore_air=false) | может оставить старые блоки цели (11 из 1469; пустой источник не ставит ничего) — для смены карты: очистка + копия без воздуха | bench/map-2 |
| `code::controller_measure_time(t, "NANOSECONDS") { ... }` | замер времени кода внутри (теперь компилируется как блок с телом; так же controller_async_run, controller_exception) | bench/cost-1.jc |
| `value::block_beneath` | целые координаты блока под ногами: `{"x":0.0,"y":3.0,"z":0.0}` | server-check-6 №6s |
| `player::is_standing_on_block` | работает и по типу (`[item("gold_block")]`), и по `locations = [value::block_beneath]` | server-check-6 №6s |
| `variable::set_item_component(item("firework_rocket"), "fireworks", "{flight_duration:1b,explosions:[{shape:\"large_ball\",colors:[I;16766720]}]}")` | работает (SNBT, массив цветов `[I;...]`); `world::launch_firework(fw, loc, "UPWARDS", instant = false)` выполняется без ошибки | server-check-6 №6f |
| `floor(75/60)`, `75 % 60`, `-7 % 60` | 1 (без .0), 15, **-7** (остаток со знаком, как в Java) | server-check-2 №8 |
| `repeat::on_range(0, 3)` | **включает** конец: 0 1 2 3 | server-check №2 |
| Локальные переменные и функции | функция **видит и меняет** локальные вызывающего кода | server-check №3, владелец |
| `%math` с текстом `"5"` | приводит к числу: `"5" * 2` = 10 | server-check №4 |
| `give_items` без `amount` | выдаёт count предмета (у `item("x")` это **1**; у `item("x", count = 3)` — 3, server-check-4 №4g) | server-check №5 |
| `potion(..., duration = 200)` | **тики** (10 секунд) | server-check №6 |
| `play_sound` без `location` | слышно игроку, без 3D-позиции (не из точки) | server-check №7 |
| `value::timestamp` | UNIX-время в **миллисекундах** | локализация меню |
| `value::world_time` | время мира в **тиках** | локализация меню |
| Единицы числовых аргументов | показаны в сигнатурах: `fade_in: number(ticks)` | названия аргументов в меню |
| `code::wait` | единица задаётся аргументом `time_unit` (по умолчанию TICKS) | данные |
| Аргументы `*_ticks` / `ticks` | тики (20 в секунду) | названия |
| `value::owner_uuid` | uuid владельца мира с дефисами, тот же вид, что `%uuid%` (`e1353378-d675-38cc-…`); у владельца `has_privilege("OWNER")` — да | server-check-27 №1 |
| `variable::parse_json` | мягкий (Gson lenient): `"oops"` → текст "oops" (не ошибка), `"[abc]"` → ["abc"], `true` → текст "true"; оборванный `"[1,"` — ошибка «java.io.EOFException: End of input», строка кода обрывается | server-check-27 №7, 9, 12 |
| `variable::to_json` | числа — с `.0` (`[6.0]`, в словаре тоже `{"a":1.0}`), вложенные списки и словари как есть, location — Java-строка `"Location{world=CraftWorld{key=…},x=0.5,y=4.0,z=0.5,pitch=0.0,yaw=0.0}"`; `=` выходит как `\u003d` | server-check-27 №9, 10 |
| `%плейсхолдер%` в ТЕКСТОВОМ значении | раскрывается сервером при использовании текста, даже если текст собран в рантайме (`replace_text("a#uuid#b", "#", "%")` → длина 38, uuid) и пришёл извне (данные custom click `"%player%"` → ник). Хранится как было (`/env var list` показывает `%player% 50%`), если пришёл как `\u0025` через parse_json | server-check-27 (+ проба pct) |
| `&` / `§` в данных custom click | доходят как есть (не цвет), `echo "&cкрасный §bсиний"` → то же | server-check-27 №12c |
| Функция с `code::wait`, вызванная из события | вызывающий ждёт её конца: `r = slow()` (wait 3 с, return 5) → 5 через 3 с | server-check-27 №8 |
| Один `player_custom_click`: jmvcc:rpc + jmvcc:cmd + свой обработчик | всё работает: status busy=1 во время сессии внутри rpc function, свой обработчик видит только чужие ключи | server-check-27 (+ проба both) |
| `trim_list(l, a, b)` (`l.slice`) | номера с **0**, конец **не** включён: [10,20,30,40,50] (1,3) → [20,30], (0,2) → [10,20], (2,2) → [], (1,5) → [20,30,40,50]; (3,1), (2,9), (-2,2) — ошибка `index_out_of_bounds` | server-check-29 (2026-11-04) |
| `trim_text(t, a, b)` (`t.slice`) | с **0**, конец не включён: "abcdef" (1,3) → "bc", (0,2) → "ab", (2,2) → ""; за пределами — **обрезается**, без ошибки: (2,9) → "cdef", (-2,3) → "abc" | server-check-29 |
| `get_char_at`, `get_index_of_subtext` (`t.index_of`), `get_list_index_of_value` | с **0**: "abcdef" char 0 → a, index_of c → 2, a → 0, нет → -1; start_index тоже с 0 ("abcabc", b, с 2 → 4); [10..50] index_of 30 → 2 | server-check-29 |
| `list_append_value(x, y)` (`x.append(y)`), y — список | **одним элементом**, как append в Python: [1,2] + [3,4] → [1,2,[3,4]], длина 3 | server-check-29 |
| `append_map(a, b)` (`a.merge(b)`), одинаковый ключ | побеждает значение **b**: {a:1,b:2} + {b:20,c:30} → {a:1,b:20,c:30} | server-check-29 |
| `list_is_empty(l)` одним получателем (`l.is_empty()`) | как [l]: [] — да, [10..50] — нет | server-check-29 |
| `code::controller_label("x") { }` / `controller_do_not_run() { }` / `controller_isolated_selection(mode) { }` — блоки с телом | label: тело **выполняется**; do_not_run: тело **не** выполняется, код после идёт; isolated_selection(EMPTY) после select::all_players: player::message в теле никому не пришло, после блока выбор снова all_players | server-check-31 (2026-11-04; прогнано под старым номером 30) |
| `variable::is_type(x, variable_type)` (`if_variable_is_type`) | список (и пустой) — **ARRAY**, словарь — **MAP**, текст (и "5") — **TEXT**, число — **NUMBER**, вектор — VECTOR, локация — LOCATION; ровно один тип «да» из шести; список в аргументе `value` **не** раскрывается (is_type(l, NUMBER) у [1,2] — нет; compare_type ALL — тоже нет); **неприсвоенная — NUMBER**; в функции параметр (копия) — тот же тип | server-check-32 (2026-11-04) |
| Проверка вида в рантайме у параметра без типа (KindInference: if is_type / else) | count(x) { x.length() } со списком / словарём / текстом → 3 / 2 / 5; x.sort() + x.length() одной проверкой — список и словарь отсортированы на месте; x.set(0, 5): список [1,2] → [5,2], словарь {k:1} → to_json {"k":1.0,"0.0":5.0} (ключ-число 0 в to_json — "0.0"); результат функции с return списка / текста — 4 / 2; параметр m: map — get_map_size | server-check-33 (2026-11-04) |

## Открытые вопросы

9. Чьё имя подставляет `%player%` в сообщении с `<all_players>` (нужно 2 игрока).
10. Кто «игрок по умолчанию» в `player_kill_player`: убийца или жертва (нужно 2 игрока).
- `entity::set_text_display_background(color, opacity)`: opacity 0..100, сервер умножает на 2.55 (255 → ошибка `Alpha[650] is not between 0-255`, проверено 2026-11-09). Вместе с `set_display_interpolation` фон меняется плавно.
- `entity::set_text_display_opacity(v)`: проценты непрозрачности (замер 2026-11-09: 10→30, 50→131, 90→232 по яркости белого на чёрном). При 0 и при v от 99 и выше текст не виден, порог между 90 и 99 не искали; зависит не от фона. С интерполяцией клиент ведёт значение как знаковый байт: через 50% она идёт не туда, нужно два этапа.
- «Не хватает места для загрузки импортируемого модуля» (проверено 2026-11-09): модуль 6,9 МБ не загрузился, 3,8 МБ и 313 строк кода загрузился (что именно считается, вес или строки, не установлено); большие данные лучше держать одной строкой и разбирать при старте (`"a\nb".split("\n")`).
- Лимит строк кода при загрузке модуля (измерено 2026-11-09, мир владельца, ранг Star): модуль из 526 строк загрузился и последняя функция выполнилась, 527 — «Не хватает места для загрузки импортируемого модуля» (600 и 1000 тоже). Строки были по одному действию (525 функций и событие, 98 КБ), так что упирается именно в число строк, а не в вес. Заливка заменяет код мира, а не добавляет: после 525 строк 526 загрузились. Почему порог 526, а игроку показано 600, не установлено; строки других видов и длинные строки отдельно не мерились.
- ЛКМ в спектаторе (проверено 2026-11-09 мостом, клавиша attack): `player_left_click`, `player_arm_swing`, `player_interact` не приходят ни свободным, ни с камерой на сущности; `player_input_event` приходит. Рабочий обход: две одинаковые `world::spawn_interaction_entity` (1×1) в одной точке, камера `spectate_target` на одной. Клик попадает в другую, сервер пересаживает камеру на неё и вызывает `player_start_spectating_entity`, это и есть клик, возвращать камеру не надо.
- Лимит строк модуля на ранге Nova (замер 2026-11-09, строки по одному действию): 548 загрузились, 570 нет, 527 проходит (на Star 526 был пределом). Одна попытка на 613 дала другую ошибку, «Импортируемый модуль повреждён», причина неизвестна. Длина строки: функция из 279 действий подряд по-прежнему «Не хватает места»; порог ниже 279 не мерился, с `--max-line 43` такие функции проходят.
- Камера на interaction (замерено по кадру 2026-11-09): глаз камеры стоит в основании сущности, а не на 0.85 высоты. Если нужна точка глаза Y, ставь interaction на Y. Пересадка между сущностями в разных точках даёт видимый прыжок вида.
- Точность чисел в выражениях (замерено 2026-11-09): `value::timestamp + 1000` дало ...403000 из ...402748, то есть результат округлён до 10 значащих цифр, и разность двух `value::timestamp` идёт с шагом около секунды. Интервалы короче секунды считать через `value::server_current_tick`, а не через миллисекунды.
