
--1. В каких городах больше одного аэропорта?
```
select city, count
from
	(select city, airport_name, count(airport_name) over (partition by city)
	from airports) t
where count > 1
group by 1, 2

-- Выбираем данные по городу, аэропорту и количеству аэропортов, которое группируем в окне по городам, из этой логики берем город и количество аэропортов
-- с условием что количество больше одного, потом сгруппировываем все это по обоим колонкам, чтоб оставить только уникальные пары значений.
```	



--2. В каких аэропортах есть рейсы, выполняемые самолетом с максимальной дальностью перелета?
```	
select distinct a.airport_name
from flights f 
join airports a on a.airport_code = f.departure_airport
where aircraft_code = 
	(select aircraft_code from aircrafts
	where range = (select max("range") from aircrafts))
	
-- Из таблицы рейсов, к которой присоединена таблица аэропортов по любому коду аэропорта (отправления или прибытия, это неважно, потому что любой аэропорт отправления
-- является и аэропортом прибытия) выбираем уникальные названия аэропортов, с условием, что код самолета, который осуществляет рейс - это самолет с максимальной
-- дальностью полета. Максимальную дальность полета выбираем через два подзапроса из таблицы самолетов.
```	
	
	
	
--3. Вывести 10 рейсов с максимальным временем задержки вылета
```		
select flight_id, flight_no, (actual_departure - scheduled_departure) as delay_time
from flights f 
where actual_departure is not null
order by 3 desc
limit 10

-- Выбираем данные по рейсам, вычитаем из времени актуального отправления время планируемого (посколько даже если актуальное отправление будет раньше планируемого,
-- результат будет отрицательный, а нам нужны только положительные и самые большие). Указываем что актуальное отправление уже состоялось (not null),
-- сортируем по результату вычитания от большего к меньшему и через limit оставляем 10 самых больших результатов.
```



--4. Были ли брони, по которым не были получены посадочные талоны?
```
select count(distinct b.book_ref)
from boarding_passes bp
	right join ticket_flights tf on tf.ticket_no = bp.ticket_no
	join tickets t on t.ticket_no = tf.ticket_no
	join bookings b on b.book_ref = t.book_ref 
where bp.ticket_no is null

-- Джойним к посадочным талонам перелеты, билеты и бронирования, потому что более короткого пути от талонов к бронированиям нет. В соединении талонов и перелетов
-- указываем правый join, потому что перелеты были точно, а талоны по ним - не факт. В результате получаем выборку с множеством значений null в колонке талонов.
-- В итоге выбираем количество уникальных позиций по бронированиям по тем строкам, где номер билета в в таблице талонов отсутствует (null).
```



-- 5. Найдите количество свободных мест для каждого рейса, их % отношение к общему количеству мест в самолете.
-- Добавьте столбец с накопительным итогом - суммарное накопление количества вывезенных пассажиров из каждого аэропорта на каждый день.
-- Т.е. в этом столбце должна отражаться накопительная сумма - сколько человек уже вылетело из данного аэропорта на этом или более ранних рейсах в течении дня.
```
select t.*, t2.sum from (
	select * from (
		with c1 as
			(select * from
				(select aircraft_code, count(seat_no) over (partition by aircraft_code)
				from seats s) t
				group by 1,2)
		select  f.flight_id,
				f.actual_departure, 
				f.departure_airport, 
				c1."count" - count(tf.ticket_no) over (partition by tf.flight_id) as free_seats,
				round((c1."count" - count(tf.ticket_no) over (partition by tf.flight_id))::numeric / c1."count"::numeric * 100, 2) as percent_free_seats
		from ticket_flights tf 
			join flights f on f.flight_id = tf.flight_id
			join aircrafts a on a.aircraft_code = f.aircraft_code 
			join c1 on c1.aircraft_code = a.aircraft_code
		where f.status = 'Departed' or f.status = 'Arrived' and c1.aircraft_code = f.aircraft_code)	t
		group by 1,2,3,4,5)
		t
	join
		(select f.flight_id,
		   		f.actual_departure, 
		   		f.departure_airport,
		   		sum(count(tf.ticket_no)) over (partition by f.departure_airport, date_trunc('day', f.actual_departure) order by f.actual_departure)
		from ticket_flights tf 
			join flights f on f.flight_id = tf.flight_id
		where f.status = 'Departed' or f.status = 'Arrived'
		group by 1, 2)
		t2
			on t2.flight_id = t.flight_id
order by 3,2

-- Создаем cte для вычисления количества мест в каждой модели самолета, к нему надо будет обратиться несколько раз. Для количества свободных мест в каждом рейсе из cte вычитаем количество
-- билетов по каждому перелету. Чтоб получить % свободных мест, результат вычитания делим на результат cte, предварительно приведя к numeric, потому что получится дробное значение.
-- Используем round чтоб убрать много цифр после запятой. Столбец с накопительным итогом пассажиров по каждому аэропорту в разрезе каждого дня присоединяем через join по id рейса, 
-- потому что по нему нельзя сделать группировку, которая была сделана в первой части задания по всем столбцам, чтоб схлопнуть результат. Для получения результата в разрезе дня использован
-- date_trunc. Также в обоих частях задания указано условие что статус рейсов "вылетел" или "прибыл", потому что работаем со свершившимся фактом.
```



--6. Найдите процентное соотношение перелетов по типам самолетов от общего количества.
```
select t.code, round(count::numeric / (select count(*) from ticket_flights tf)::numeric * 100, 1) as "percent"
	from
	(select f.aircraft_code as code, count(*) over (partition by f.aircraft_code)
from ticket_flights tf
join flights f on f.flight_id = tf.flight_id) t
group by 1, 2

-- Из таблицы перелетов считаем в окне количество перелетов по каждому коду самолета, заворачиваем все это в подзапрос, чтоб сгруппировать полученные данные,
-- в результате выводим id самолетов и количество перелетов по каждому, деленное на количество всех перелетов, перед делением каждый пункт приводим к numeric,
-- что в результате получить дробь, а не ноль, округляем оператором round, чтоб убрать кучу цифр после запятой.
```



--7. Были ли города, в которые можно  добраться бизнес - классом дешевле, чем эконом-классом в рамках перелета?
```
with c1 as 
		(select tf.amount, f.flight_id, f.departure_airport, f.arrival_airport 
			from ticket_flights tf 
			join flights f on tf.flight_id = f.flight_id
			where tf.fare_conditions = 'Economy'
		group by 2,1),
	c2 as
		(select tf.amount, f.flight_id, f.departure_airport, f.arrival_airport
			from ticket_flights tf 
			join flights f on tf.flight_id = f.flight_id
			where tf.fare_conditions = 'Business'
		group by 2,1),
	c3 as 
		(select airport_code, city
			from airports)
select distinct c3.city
	from c1
	join c2 on c2.flight_id = c1.flight_id
	join c3 on c3.airport_code = c2.arrival_airport
	where c2.amount < c1.amount
		
	-- Создаем три cte. Первые два - это выборка данных из таблиц перелетов и рейсов с условием что класс перелета "эконом" или "бизнес", группируем по сумме
	-- платежа и id рейса, чтоб схлопнуть по уникальным строкам. Третье cte - выборка кодов аэропорта и города, чтоб в итоговом результате получить именно
	-- город, а не аэропорт. Джойним первые 2 cte по id рейса, третье cte джойним по аэропорту прибытия (потому что вопрос звучал "города, В КОТОРЫЕ можно
	-- добраться"). В условии указываем что платеж из 2 cte меньше платежа из 1 cte. Выводим уникальные названия городов из третьего cte, чтоб убрать
	-- возможные дублирующиеся строки. В результате получаем пустую таблицу, следовательно таковых городов нет.
```


	
--8. Между какими городами нет прямых рейсов?
```	
create view view_1 as
	(select distinct a.city as dep_city, a2.city as arr_city
		from flights f
	join airports a on a.airport_code = f.departure_airport
	join airports a2 on a2.airport_code = f.arrival_airport)

create view view_2 as
	(select city as dep_city
		from airports)

create view view_3 as
	(select city as arr_city
		from airports)

select dep_city, arr_city
	from view_2, view_3
except
select *
	from view_1
where dep_city != arr_city

-- Создаем представления, как было указано в задаче. Первое - с уникальными комбинациями всех существующих перелетов, из таблицы рейсов, и также на все города
-- отправления и прибытия отдельно, из таблицы аэропортов. После перемножаем представления с городами отправления и прибытия через декартово произведение, чтоб
-- получить все вероятно возможные перелеты, и вычитаем из этого с помощью оператора Except все существующие перелеты в из первого представления.
-- В итоге получаем все несуществующие перелеты, что в нашем случае равно условию задачи. Также указываем что город отправления не равен городу прибытия.
```



--9. Вычислите расстояние между аэропортами, связанными прямыми рейсами, сравните с допустимой максимальной дальностью перелетов  в самолетах, обслуживающих эти рейсы.
```
with c1 as
	(select distinct a1.airport_name as dep_airport,
			a2.airport_name as arr_airport,
			a."range",
			round (acos(sind(a1.latitude) * sind(a2.latitude) + cosd(a1.latitude) * cosd(a2.latitude) * cosd(a1.longitude - a2.longitude))::numeric, 2) * 6371 as distance
	from flights f 
	join airports a1 on a1.airport_code = f.departure_airport
	join airports a2 on a2.airport_code = f.arrival_airport
	join aircrafts a on a.aircraft_code = f.aircraft_code)
select c1.*,
		(case when c1."range" > c1.distance then 'ok'
		else 'break'
		end) as distance_test
from c1

-- Из таблицы рейсов выбираем уникальные сочетания аэропортов, чтоб получить все существующие прямые рейсы, присоединяем дальность полета самолета, обслуживающего
-- данный рейс, и в последней колонке по заранее известной формуле вычисляем расстояние между этими аэропортами, используя операторы sind и cosd. Чтоб проверить, долетит ли
-- самолет, заворачиваем все вычисления в cte, получаем оттуда все данные и добавляем еще колонку, где через case сравниваем дальность полета самолета и полученное расстояние
-- между аэропортами.
```
