# Типы связей в реляционной базе данных

## Введение

Связи между таблицами в базе данных — это основа хранения данных в [СУБД](https://ru.wikipedia.org/wiki/%D0%A1%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%B0_%D1%83%D0%BF%D1%80%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D1%8F_%D0%B1%D0%B0%D0%B7%D0%B0%D0%BC%D0%B8_%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85). 
Связи позволяют [нормализировать БД](https://docs.microsoft.com/ru-ru/office/troubleshoot/access/database-normalization-description) и настроить отношение между данными таблиц и делать эффективные выборки данных. Кроме того, настроенные связи позволяют отслеживать изменения в связанных таблицах и применять осбобые действия к данным.

Ниже рассмотрим основные концепции связей: [Foreign Key](#Foreign-Key) и [JOINs](#JOINs). 

## Foreign Key 

Связь мужду таблицами организуется через внешний ключ (foreign key). Данный ключ связывает поле (значение) исходной таблицы с Primary Key внешней таблицы. 
Через внешний ключ можно производить не только выборку данных, но и контролировать удаление данных в главной таблице:
- NO ACTION не производит никаких действий;
- SET NULL зависимые данные установятся в NULL при удалении записи из главной главной таблицы (primary table);
- CASCADE удаляются зависимые данные. Опасная операция. В реальной жизни используется редко;

Описание шаблона добавления внешнего ключа ([]link](https://docs.microsoft.com/en-us/sql/relational-databases/tables/create-foreign-key-relationships?view=sql-server-ver16))

```sql
ALTER TABLE <table_name>
ADD CONSTRAINT FK_<primary_table_name>_<primary_table_column> FOREIGN KEY (<primary_table_column>)
REFERENCES <ref_table_name>(<table_pk>)
ON DELETE <type>
ON UPDATE <type>
```

## JOINs

Установленные отношения между таблицами позволяют сделать выборки данных из связанных этих таблиц. Существует несоклько механизмов соединения двух таблиц в запросе. Например, Oracle содержит Natural Join, который соединяет колонки с одинаковыми именами в таблицах (используется крайне редко). Но, хотелось бы отметить, что основные типы JOINs совпадают для всех реляционных СУБД. 

Рассмотрим основные типы JOINs для MS SQL SERVER.

Будем считать что у нас есть левая и правая таблицы которые соединяем через JOIN (легче понять предположив что левая и правая относительно слова JOIN).

### LEFT (OUTER) JOIN

![Left Join](img/leftjoin.png?raw=true "Left Join")

Всегда выводить данные по левой таблице. Если правая таблица не содержит связанных данных, то выводить NULL для этих значений.

```sql
select * 
from A left join B on A.ID = B.A_ID
```
A1  | B1   
A2  | B2   
A3  | NULL  

### RIGHT (OUTER) JOIN

![Right Join](img/rightjoin.png?raw=true "Right Join")

Обратное от [Left Join](#LEFT-(OUTER)-JOIN). Используется крайне редко и на практике обычно не приветствуется. Всегда можно переписать на Left Join (читается запрос легче). В частных случаях бывает что Right Join дает лучшую статистику выполнения и имеет место быть в целях оптимизации запроса. Но бывает это крайне редко.

A left join B = B right join A

```sql
select * 
from B right join A on A.ID = B.A_ID
```
A1  | B1   
A2  | B2   
A3  | NULL  

### INNER JOIN 

![Inner Join](img/innerjoin.png?raw=true "Inner Join")

Выводить значения для строки из левой таблицы только если есть связанные данные в правой таблице. Используется очень часто чтобы отфильтровать данные левой таблицы и выводить только те записи по которым есть, значения в правой. 

```sql
select * 
from A inner join B on A.ID = B.A_ID

-- аналог запроса на left join
select * 
from A left join B on A.ID = B.A_ID
where B.ID is not null
```

A1  | B1 
A2  | B2 

### CROSS JOIN 

Это пересечение всех строк из левой таблицы со всеми строками правой таблицы. 

```sql
select * 
from A cross join B
```

A1  | B1 
A1  | B2 
A2  | B1 
A2  | B2 

### FULL JOIN 

![Full Join](img/fulljoin.png?raw=true "Full Join")

Легче всего представить что это смешанное сочетание Left Join и Right Join. Вначале выводятся значения левой таблицы, а правой заполняются NULL, затем наоборот.

Запрос выводит сначала пересечение значений, если нет пересечений то выводит значения по A и B c NULL

```sql
select * 
from A full join B on A.ID = B.A_ID
```

A1  | B1   
A2  | B2   
A3  | NULL 
NULL| B4   


## Типа отношений между таблицами 

Понимая что такое [Foreign Key](#Foreign-Key) и [JOINs](#JOINs) создадим реальный пример бизнес задачи и рассмотрим найстройку связей между таблицами.
Введем сущности Clinics, Doctors, Patients и Appointments. 

- Докор работает (или нет) только в одной клинике. 
- У доктора может быть вышестоящий менеджер. 
- Пациент может обращаться в разные клиники, к разным докторам. 

Создадим таблицы БД пока без связей c [сурогатными](#https://ru.wikipedia.org/wiki/%D0%A1%D1%83%D1%80%D1%80%D0%BE%D0%B3%D0%B0%D1%82%D0%BD%D1%8B%D0%B9_%D0%BA%D0%BB%D1%8E%D1%87) Primary Keys


```sql
CREATE TABLE dbo.Clinics (
	ID int,
	Name varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
	CONSTRAINT Clinics_PK PRIMARY KEY (ID)
);


CREATE TABLE dbo.Patients (
	ID int,
	Name varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
	CONSTRAINT Patients_PK PRIMARY KEY (ID)
);


CREATE TABLE dbo.Doctors (
	ID int,
	Name varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
	Clinic_ID int NULL,
	Manager_ID int NULL,
	CONSTRAINT Doctors_PK PRIMARY KEY (ID)
);


CREATE TABLE dbo.Appointments (
	ID int,
	[Date] datetime2(0) NOT NULL,
	Patient_ID int NOT NULL,
	Doctor_ID int NOT NULL,
	CONSTRAINT Appointments_PK PRIMARY KEY (ID)
);
```

Фейковые данные 

```sql
insert into dbo.Clinics values (1, 'First Clinic'), (2, 'Second Clinic')
insert into dbo.Doctors values (10, 'Doctor', 1, NULL), (11, 'Assistent Doctor', 1, 10), (12, 'Another Doctor', 2, NULL), (13, 'Retired Doctor', NULL, NULL), (15, 'Assist 2 Doctor', 1, 11)
insert into dbo.Patients values (100, 'First Patient'), (101, 'Second Patient')
insert into dbo.Appointments values (1000, GETDATE(), 100, 10), (1001, GETDATE(), 101, 10), (1002, GETDATE(), 100, 12) 
```



### Отношения «один к одному»

Использовать данную связь когда значению из таблицы может соответствовать только одна запись из внешней таблицы.
Доктор может работать только в одной клинике. Соответственно, можем предположить что связь между Clinics и Doctors будет «один к одному».

```sql
ALTER TABLE dbo.Doctors ADD CONSTRAINT FK_Doctors_ClinicID FOREIGN KEY (Clinic_ID) REFERENCES dbo.Clinics(ID);
```

```sql
select 
	d.Name,
	c.Name 
from 
	dbo.Doctors d 
	left join dbo.Clinics c on c.ID  = d.Clinic_ID;
```

### Отношение «один ко многим»

Одной записи из таблицы соответствует несколько записей из внешней. Данная связь является очень распространенной при построении схемы БД. 

Хотя доктор может принадлежать только одной клинике, клиники в свою очередь сожержат штат докторов. Это отношение «один ко многим».

```sql
select 
	c.Name,
	d.Name 
from 
	dbo.Clinics c 
	inner join dbo.Doctors d on d.Clinic_ID = c.ID;
```
CLINIC         |DOCTOR 
---            | ---
First Clinic   |Doctor             
First Clinic   |Assistent Doctor   
First Clinic   |Assist 2 Doctor    
Second Clinic  |Another Doctor     

### Отношение «многие ко многим» 

Организуется через промежуточную таблицу, в которой есть внешние ключи на разные таблицы.

В таблице Appointments есть связь на таблицы Doctors и Patients. Таким образом, организована связь между пациентами и докторами. Пациент может посещать несколько докторов, а доктора принимать несколько пациентов. 

```sql
ALTER TABLE dbo.Appointments ADD CONSTRAINT FK_Appointments_DoctorID FOREIGN KEY (Doctor_ID) REFERENCES dbo.Doctors(ID);
ALTER TABLE dbo.Appointments ADD CONSTRAINT FK_Appointments_PatientID FOREIGN KEY (Patient_ID) REFERENCES dbo.Patients(ID);
```

Хотел бы отметить, что это довольно распространенный вопрос на собеседовании для SQL developer. Если  программист может на примере объяснить как строится связь «многие ко многим», то это уже хороший индикатор для интервьюера.

Получить список посещений пользователя с указанием клиники и докторов.
```sql
select
	p.Name,
	a.[Date],	
	d.Name,
	c.Name
from 
	dbo.Patients p 
	inner join dbo.Appointments a on a.Patient_ID = p.ID 
	inner join dbo.Doctors d on d.ID = a.Doctor_ID 
	inner join dbo.Clinics c on c.ID = d.Clinic_ID 
where
	p.ID = 100;

```

|PATIENT|DATE|DOCTOR|CLINIC|
|----|----|----|----|
|First Patient|2022-08-06 07:41:45.000|Doctor|First Clinic|
|First Patient|2022-08-06 07:41:45.000|Another Doctor|Second Clinic|


### Связь с самим собой 

Такой тип связи называется рекурсивным или иерархическим. Связывание строки со строкой из той же таблицы. Полезно при отображении древовидной структуры. 

Таблица Doctors содержит колонку Manager. В которой указано кто из докторов является менеджером текущего доктора. Здесь связь на строку из той же таблицы докторов.

Как рекурсивно получить список докторов, у которых определенный доктор является вышестоящим менеджером.  

```sql
with cte as (
	select 
		d.ID,
		d.Name 
	from 
		dbo.Doctors d 
	where 
		d.ID = 10
	union all 
	select 
		d2.ID,
		d2.Name 
	from 
		dbo.Doctors d2 
		inner join cte on cte.ID = d2.Manager_ID  
)
select 
	*
from 
	cte 
```

## Выводы

Таблицы БД без связей, являются просто набором данных который ограничен в своем применении. Это становится просто набором данных без какой либо возможности сделать эффективный запрос на выборку значений. Когда происходит построение связей, сервер СУБД производит построение индексов и оптимизирует выборку данных из хранилища тем самым давая возможность получать значения из связанных таблиц максимально быстро и самым эффективным способом. 
Помимо запросов, наличие связей позволяет контролировать что данные в хранилище не будут повреждены и останутся консистентными. 
Правильно построенные связи во многом описывают бизнес-модель на уровне хранения данных и очень важны для построения всего ПО. 