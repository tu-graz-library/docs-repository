# pgAdmin

The pgAdmin package is a free and open-source graphical user interface (GUI) administration tool for PostgreSQL, which is configured as a service for your development.

docker-services.yml
```yml
  pgadmin:
    image: dpage/pgadmin4
    ports:
      - "5050:5050"
    environment:
      - PGADMIN_DEFAULT_EMAIL=<your_email_address>
      - PGADMIN_DEFAULT_PASSWORD=<your_password>
      - PGADMIN_LISTEN_PORT=5050
```
docker-compose.yml
```yml
  pgadmin:
    extends:
      file: docker-services.yml
      service: pgadmin
```
You can easily access pgAdmin 4 from your web browser.

visit [http://localhost:5050](). You should see the pgAdmin login page. Login with your **email** and **password**.


![](images/pgadmin-login.png?raw=true)


Once you login, you should see the pgAdmin dashboard.

![](images/pgadmin-dashboard.png?raw=true)

Now, to add the PostgreSQL server running as a Docker container, right click on Servers, and then go to Create > Server…

![](images/pgadmin-create-s.png?raw=true)

In the General tab, type in your server Name.

![](images/pgadmin-create-server-step1.png?raw=true)


Now, go to the Connection tab and type in ```db``` as Host name/address, **(db is the container name for PostgreSQL)** 5432 as Port, postgres as Maintenance database, admin as Username, secret as Password and check Save password? checkbox. Then, click on Save.

NOTE: Ad ```Host name/address``` add docker container name or service name of your database.
in real production it can be the IP of address of Database.

![](images/pgadmin-create-server-step2.png?raw=true)


pgAdmin 4 should be connected to your PostgreSQL database. Now, you can work with your PostgreSQL database as much as you want.

![](images/pgadmin-connected.png?raw=true)
