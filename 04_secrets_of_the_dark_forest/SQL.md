-- Задача 1. Исследование доли платящих игроков

-- 1.1. Доля платящих пользователей по всем данным:

SELECT COUNT (id) AS total_players, -- посчитали общее количество игроков
SUM (payer) AS total_paying_players, -- общее количество платящих игроков
ROUND ((AVG (payer)*100), 2) AS perc_of_paying_players --доля платящих игроков, умноженная на 100 и округленная до двух знаков после точки
FROM fantasy.users;

-- 1.2. Доля платящих пользователей в разрезе расы персонажа:

SELECT u.race_id,
r.race,
SUM (u.payer) AS total_paying_players, -- общее количество платящих игроков
COUNT (u.id) AS total_players, --общее количество игроков
ROUND(SUM(u.payer::numeric)*100 / COUNT(u.id), 2) AS perc_payer_players_by_race -- вычислили долю платящих игроков в разрезе каждой расы, округлив результат до двух знаков после точки
FROM fantasy.users AS u 
LEFT JOIN fantasy.race AS r USING (race_id) 
GROUP BY u.race_id, r.race
ORDER BY perc_payer_players_by_race DESC; -- сортируем по доле платящих игроков в порядке убывания

-- Задача 2. Исследование внутриигровых покупок
-- 2.1. Статистические показатели по полю amount:

SELECT COUNT (transaction_id) AS total_transaction, --общее количество покупок эпических предметов
COUNT(transaction_id) FILTER (WHERE amount = 0) total_zero_transaction, -- столбец с количеством "нулевых покупок"
SUM (amount) AS total_amount, -- суммарная стоимость покупок
MIN (amount) AS min_amount, -- минимальная стоимость покупки
MAX (amount) AS max_amount, -- максимальная стоимость покупки
ROUND (AVG (amount::numeric), 2) AS avg_amount, -- средняя стоимость покупки, округленная до двух знаков после точки
PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY amount) AS median_amount, -- медиана стоимости покупок
ROUND (STDDEV (amount::numeric), 2) AS stddev_amount -- стандартное отклонение 
FROM fantasy.events; --изменила integer на numeric

-- 2.2: Аномальные нулевые покупки:

WITH cte AS (
SELECT COUNT (amount) AS total_purchases,
	(SELECT COUNT (amount) AS total_zero_purchases
	FROM fantasy.events
	WHERE amount = 0) AS total_zero_purchases -- в подзапросе вычислим, сколько покупок было совершено за 0 лепестков (аномальное значение)
FROM fantasy.events
) -- создаем СТЕ, чтобы определить количество покупок всего и количество покупок, где стоимость покупки за райские лепестки равна 0
SELECT total_purchases,
total_zero_purchases,
total_zero_purchases::numeric / total_purchases AS perc 
FROM cte; -- в основном запросе рассчитаем долю покупок с нулевой стоимостью, приведя числитель к типу данных NUMERIC

-- 2.3: Сравнительный анализ активности платящих и неплатящих игроков:

WITH paying_players AS (
	SELECT u.id AS id,
		CASE
			WHEN u.payer = 1
				THEN 'платящий'
			ELSE 'неплатящий'
		END AS payer_status,
	COUNT (e.transaction_id) AS total_purchases, --общее количество покупок
	SUM (e.amount) AS total_amount -- общая сумма покупок
	FROM fantasy.users AS u 
	LEFT JOIN fantasy.events AS e USING (id)
	WHERE e.amount > 0
	GROUP BY u.id, payer_status
) -- создаем СТЕ, чтобы рассчитать количество покупок и сумму покупок для каждой категории игроков
SELECT payer_status,
COUNT (id) AS total_players, -- общее количество игроков
ROUND (AVG (total_purchases::INT), 2) AS avg_total_purchases, --среднее количество покупок
ROUND (AVG (total_amount::INT), 2) AS avg_total_amount --средняя суммарная стоимость покупок
FROM paying_players
GROUP BY payer_status;
-- В основном запросе на основе результата СТЕ считаем общее количество игроков, среднее количество покупок и среднюю стоимость покупок, округлив результаты до двух знаков после точки

-- 2.4: Популярные эпические предметы:

WITH epic_item_sales AS (
	SELECT e.item_code,
	i.game_items,
	COUNT (e.transaction_id) AS total_sales,
	COUNT (DISTINCT id) AS total_players
	FROM fantasy.events AS e 
	LEFT JOIN fantasy.items AS i USING (item_code) --присоединили таблицу items для наглядности результата
	WHERE amount > 0
	GROUP BY e.item_code, i.game_items
) --создаем СТЕ, в котором рассчитали общее количество продаж по каждому эпическому предмету, а также общее количество уникальных игроков для дальнейших расчетов
SELECT item_code,
game_items,
SUM (total_sales) AS sum_total_sales,
ROUND ((SUM (total_sales)::NUMERIC) * 100 / (SELECT SUM (total_sales) FROM epic_item_sales), 2) AS perc_sales_by_epic_item,
ROUND ((SUM (total_players)::NUMERIC) * 100 /(SELECT count(DISTINCT id) FROM fantasy.events WHERE amount > 0), 2) AS perc_sales_by_players
FROM epic_item_sales
GROUP BY item_code, game_items
ORDER BY perc_sales_by_players DESC;
-- в общем запросе рассчитали сумму всех продаж, а также процент продаж эпического предмета относительно общего количества продаж и долю игроков, которые хотя бы раз покупали этот предмет
-- чтобы высчитать долю от общей суммы всех продаж эпических предметов и долю игроков, в знаменателе были использованы подзапросы.

-- Часть 2. Решение ad hoc-задач
-- Задача 1. Зависимость активности игроков от расы персонажа:

WITH total_players_by_race AS ( -- в этом СТЕ считаем общее количество зарегистрированных игроков для каждой расы
	SELECT race_id,
	COUNT (id) AS total_players -- всего зарегистрировано игроков
	FROM fantasy.users
	GROUP BY race_id
	ORDER BY total_players DESC
),
players_with_purchase_by_race AS ( -- в этом СТЕ считаем количество игроков, которые совершают внутриигровые покупки для каждой расы
	SELECT u.race_id,
	COUNT (DISTINCT u.id) AS total_players_with_purchase --всего игроков, совершивших покупки
	FROM fantasy.users AS u 
	LEFT JOIN fantasy.events AS e USING (id)
	WHERE amount > 0
	GROUP BY u.race_id
),
payer_players_with_purchase_by_race AS ( -- в этом СТЕ считаем количество платящих игроков, которые совершают внутриигровые покупки для каждой расы
	SELECT race_id,
	COUNT (DISTINCT id) AS total_payer_players_with_purchase --всего платящих игроков, совершивших покупки
	FROM fantasy.users
	WHERE payer = 1
	GROUP BY race_id
),
player_activity_by_race AS ( --в этому СТЕ собираем информацию об активности игроков с учётом расы персонажа
	SELECT u.race_id,
	COUNT (DISTINCT e.id) FILTER (WHERE e.amount > 0) AS total_players_with_purchase,
	COUNT (DISTINCT e.id) FILTER (WHERE u.payer = 1) AS total_payer_players_with_purchase, 
	COUNT (e.transaction_id) AS total_purchases, -- количество покупок
	SUM (e.amount) AS total_amount, --сумма покупок
	ROUND (AVG (e.amount::numeric), 2) AS avg_amount -- средняя стоимость одной покупки на одного игрока
	FROM fantasy.users AS u 
	LEFT JOIN fantasy.events AS e USING (id)
	GROUP BY u.race_id
)
SELECT cte1.race_id,
r.race,
cte1.total_players,
cte2.total_players_with_purchase,
cte3.total_payer_players_with_purchase,
--далее считаем долю игроков, совершивших покупки, от общего количества игроков
ROUND ((cte4.total_payer_players_with_purchase::NUMERIC * 100 / cte4.total_players_with_purchase), 2) AS perc_players_with_purchase_by_race,
--далее считаем долю платящих игроков от количества игроков, совершивших покупки
ROUND ((cte3.total_payer_players_with_purchase::NUMERIC * 100 / cte4.total_purchases), 2 ) AS perc_of_payer_players_by_race,
-- далее считаем среднее количество покупок на одного игрока
ROUND ((cte4.total_purchases::numeric / cte4.total_players_with_purchase), 2) AS avg_count_purchases_by_player,
cte4.avg_amount AS avg_amount_by_player,
-- и наконец считаем среднюю суммарную стоимость всех покупок на одного игрока
ROUND ((cte4.total_amount::numeric / cte4.total_players_with_purchase), 2) AS avg_sum_amount_by_player
FROM total_players_by_race AS cte1 
--присоединяем все СТЕ и таблицу с наименованиями рас для наглядности
LEFT JOIN fantasy.race AS r USING (race_id)
LEFT JOIN players_with_purchase_by_race AS cte2 USING (race_id)
LEFT JOIN payer_players_with_purchase_by_race AS cte3 USING (race_id)
LEFT JOIN player_activity_by_race AS cte4 USING (race_id)
ORDER BY total_players DESC; -- сотируем по количеству игроков

