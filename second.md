# Домашнее задание #2

### Ход работы

1. Установим Docker
2. Docker Engine установлен вместе с docker
3. Сделаем каталог `C:\var\lib\postgres`
4. Развернем контейнер PostgreSQL 15, смонтировав в него созданный ранее каталог
   ```
   docker run -it --name psql-container -p 5432:5432 -e POSTGRES_PASSWORD=postgres -d -v C:\var\lib\postgres:/var/lib/postgresql/data postgres
   ```
   После старта контейнера увидим, что директория для хранения данных была заполнена, а в визуальном интерфейсе Docker-а
   были отмечены смонтированные файлы значком **MOUNT**

   ![Screenshot 2023-07-16 130700.png](images%2FScreenshot%202023-07-16%20130700.png)
5. Развернем новый контейнер с клиентом postgres
   ```
   docker run -it --name psql-container-client -p 5433:5432 -e POSTGRES_PASSWORD=postgres -d postgres
   ```
6. подключимся из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
    - Для подключения используем `host.docker.internal:5432`
       ```shell
       psql postgresql://postgres:postgres@host.docker.internal:5432/postgres
       ```
    - Создадим таблицу, добавим записей, получим записи для проверки.
       ```sql
       create table test(number int);
       INSERT INTO test(number) SELECT generate_series(1,1000);
       SELECT * FROM test LIMIT 3;
       ```

      ![Screenshot 2023-07-16 133253.png](images%2FScreenshot%202023-07-16%20133253.png)
7. Далее подключимся к `psql-container` извне. Например, с виртуальной машины. Для этого необходимо знать общедоступный
   ip адрес и открытый порт на котором находится контейнер, также нужно знать пароль и логин, кроме того необходим
   инструмент для подключения (например postgres установленный)
      ```shell
      'C:\Program Files\PostgreSQL\15\bin\psql.exe' -p 5432 -U postgres -h put_ip -d postgres -W
      ```
8. Удалим контейнер с сервером `psql-container`
   ```shell
   docker stop psql-container
   docker rm psql-container
   ```
9. Создадим контейнер `psql-container` повторно с помощью команды п.4
10. Подключимся из `psql-container-client` в бд контейнера `psql-container` повторно с помощью команды п.6
11. Проверим, что данные остались на месте, для этого выполним запрос SQL
      ```sql
      SELECT * FROM test LIMIT 3;
      ```
    Данные остались на своем месте

    ![Screenshot 2023-07-16 135338.png](images%2FScreenshot%202023-07-16%20135338.png)
