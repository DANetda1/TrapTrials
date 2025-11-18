# Ответы на задания семинара 6

## Часть 1: Select

### Вопрос 1
Показать все названия книг вместе с именами издателей.

```sql
SELECT b.Title,
       p.PubName
FROM Book b
LEFT JOIN Publisher p
    ON b.Publisher_Name = p.PubName;
```

### Вопрос 2
В какой книге наибольшее количество страниц?

```sql
SELECT Title,
       Number_of_pages
FROM Book
WHERE Number_of_pages = (
    SELECT MAX(Number_of_pages)
    FROM Book
);
```

### Вопрос 3
Какие авторы написали более 5 книг?

```sql
SELECT Author
FROM Book
GROUP BY Author
HAVING COUNT(*) > 5;
```

### Вопрос 4
В каких книгах более чем в два раза больше страниц, чем среднее количество страниц для всех книг?

```sql
SELECT Title,
       Number_of_pages
FROM Book
WHERE Number_of_pages > 2 * (
    SELECT AVG(Number_of_pages)
    FROM Book
);
```

### Вопрос 5
Какие категории содержат подкатегории?

```sql
SELECT DISTINCT c.CategoryName
FROM Category c
WHERE EXISTS (
    SELECT 1
    FROM Category sub
    WHERE sub.ParentCat = c.CategoryName
);
```

### Вопрос 6
У какого автора (предположим, что имена авторов уникальны) написано максимальное количество книг?

```sql
SELECT Author
FROM Book
GROUP BY Author
ORDER BY COUNT(*) DESC
LIMIT 1;
```

### Вопрос 7
Какие читатели забронировали все книги (не копии), написанные "Марком Твеном"?

```sql
SELECT r.LastName, r.FirstName
FROM Reader r
WHERE EXISTS (SELECT 1 FROM Book WHERE Author = 'Марк Твен')
  AND NOT EXISTS (
        SELECT 1
        FROM Book b
        WHERE b.Author = 'Марк Твен'
          AND NOT EXISTS (
              SELECT 1
              FROM Borrowing br
              WHERE br.ID = r.ID
                AND br.ISBN = b.ISBN
          )
  );
```

### Вопрос 8
Какие книги имеют более одной копии?

```sql
SELECT b.Title,
       c.ISBN,
       COUNT(*) AS copies_count
FROM Copy c
JOIN Book b
    ON b.ISBN = c.ISBN
GROUP BY b.Title, c.ISBN
HAVING COUNT(*) > 1;
```

### Вопрос 9
ТОП 10 самых старых книг

```sql
SELECT Title,
       Year
FROM Book
ORDER BY Year
LIMIT 10;
```

### Вопрос 10
Перечислите все категории в категории "Спорт" (с любым уровнем вложености).

```sql
WITH RECURSIVE CategoryTree AS (
    SELECT c.CategoryName,
           c.ParentCat
    FROM Category c
    WHERE c.CategoryName = 'Спорт'

    UNION ALL

    SELECT c2.CategoryName,
           c2.ParentCat
    FROM Category c2
    JOIN CategoryTree ct
      ON c2.ParentCat = ct.CategoryName
)
SELECT CategoryName
FROM CategoryTree;
```

## Часть 2: Insert / Update / Delete

### Вопрос 1
Добавьте запись о бронировании читателем 'Василеем Петровым' книги с ISBN 123456 и номером копии 4.

```sql
INSERT INTO Borrowing (ID, ISBN, CopyNumber, ReturnDate)
SELECT r.ID,
       '123456' AS ISBN,
       4        AS CopyNumber,
       NULL     AS ReturnDate
FROM Reader r
WHERE r.LastName = 'Петров'
  AND r.FirstName = 'Василий';
```

### Вопрос 2
Удалить все книги, год публикации которых превышает 2000 год.

```sql
DELETE FROM Book
WHERE Year > 2000;
```

### Вопрос 3
Измените дату возврата для всех книг категории "Базы данных", начиная с 01.01.2016, чтобы они были в заимствовании на 30 дней дольше (предположим, что в SQL можно добавлять числа к датам).

```sql
UPDATE Borrowing b
SET ReturnDate = ReturnDate + INTERVAL '30 days'
WHERE ReturnDate >= DATE '2016-01-01'
  AND b.ISBN IN (
      SELECT bc.ISBN
      FROM BookCategory bc
      WHERE bc.CategoryName = 'Базы данных'
  );
```

## Часть 3: Интерпретация запросов

### Запрос 1

```sql
SELECT s.Name, s.MatrNr FROM Student s
WHERE NOT EXISTS (
SELECT * FROM Check c WHERE c.MatrNr = s.MatrNr AND c.Note >= 4.0 ) ;
```

**Опишите на русском языке результат запроса выше:**

Запрос выбирает имена и номера тех студентов, у которых нет ни одной записи в таблице "Check" с "Note" >= 4.0

### Запрос 2
```sql
( SELECT p.ProfNr, p.Name, sum(lec.Credit)
FROM Professor p, Lecture lec
WHERE p.ProfNr = lec.ProfNr
GROUP BY p.ProfNr, p.Name)
UNION
( SELECT p.ProfNr, p.Name, 0
FROM Professor p
WHERE NOT EXISTS (
SELECT * FROM Lecture lec WHERE lec.ProfNr = p.ProfNr ));
```

**Опишите на русском языке результат запроса выше:**

Запрос возвращает всех профессоров и суммарное кол-во кредитов по их лекциям. Сперва берутся профессоры, которые читают минимум 1 лекцию и считается сумма "Credit" по их лекциям. Если же у профессора нет ни одной лекции, то для них в сумму "Credit" подставляется 0. Результат это список все профессоров с их общим числом кредитов, подсчитанные их лекциями.

### Запрос 3
```sql
SELECT s.Name, p.Note
FROM Student s, Lecture lec, Check c
WHERE s.MatrNr = c.MatrNr AND lec.LectNr = c.LectNr AND c.Note >= 4
AND c.Note >= ALL (
SELECT c1.Note FROM Check c1 WHERE c1.MatrNr = c.MatrNr )
```

**Опишите на русском языке результат запроса выше:**

Запрос выбирает имена студентов и их оценки по лекциям. Для студентов берутся строки из "Check", в которых "Note" >= 4.0 и этот "Note" должен быть не меньше всех остальных "Note". Результат это студенты и их максимальные оценки, >=4 по лекциям.
