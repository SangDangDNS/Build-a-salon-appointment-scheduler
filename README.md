# Build-a-salon-appointment-scheduler

For this project, I will create an interactive Bash program that uses PostgreSQL to track the customers and appointments for the salon.

## Step 1: Create Docker-compose file

You will use docker compose to create a container docker for Postgres.  

Pls create a file `docker-compose.yaml` and a folder `salon_data`.    

File `docker-compose.yaml`    
```
services:
  pgdatabase:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=salon
    volumes:
      - "./salon_data:/var/lib/postgresql/data:rw"
    ports:
      - "5432:5432"
```

To start a postgres instance, run this command:  
`sudo docker-compose up -d`

**Note:** If you want to stop that docker compose, pls enter this command: `sudo docker-compose down`  

Ensure that the .pgpass file is properly set up to avoid any password prompts. If the .pgpass file doesn't exist, create it in your home directory and set the appropriate permissions:

```
touch ~/.pgpass
chmod 600 ~/.pgpass
```

Open the .pgpass file in a text editor and add the following line with the appropriate values for your PostgreSQL server:

```
localhost:5432:salon:root:your_password_here
``` 

To log in to PostgreSQL with psql. Do that by entering this command in your terminal:

```
psql -h <hostname> -p <port> -U <username> -d <database>
```

## Step 2: Create table in DB

Create 3 tables `services`, `customers` and `appointments` for DB like the below:

```
CREATE TABLE services (
    service_id SERIAL PRIMARY KEY,
    name VARCHAR(30)
);
```

```
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(30),
    phone VARCHAR(20) NOT NULL UNIQUE
);
```

```
CREATE TABLE appointments (
    appointment_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL REFERENCES customers(customer_id),
    service_id INT NOT NULL REFERENCES services(service_id),
    time varchar(20)
);
```

Next, insert some data to table `services`

```
INSERT INTO services (name) VALUES ('cut'),('color'),('perm'),('style'),('trim');
```

## Step 3: Create Bash file

And then, you will create the Bash script file `salon.sh`. Ensure the script has execution permission: 

```
chmod +x salon.sh
```

File `salon.sh`  

```
#!/bin/bash

# Set PGPASSFILE environment variable to point to the .pgpass file
export PGPASSFILE=/home/sang/.pgpass

PSQL="psql -h localhost -p 5432 -U root -d salon --no-align --tuples-only -c"

echo -e "\n~~~~~ MY SALON ~~~~~\n"
echo -e "Welcome to My Salon, how can I help you?\n" 

MAIN_MENU() {
  if [[ $1 ]]
  then
    echo -e "\n$1"
  fi
  SERVICES=$($PSQL "SELECT * FROM services")
  # display services
  echo "$SERVICES" | while IFS='|' read -r SERVICE_ID NAME
  do
    echo "$SERVICE_ID) $NAME"
  done
  # input for service
  read SERVICE_ID_SELECTED

  case $SERVICE_ID_SELECTED in
    [1-5]) APPOINTMENT ;;
        *) MAIN_MENU "I could not find that service. What would you like today?" ;;
  esac
}

APPOINTMENT() {
  echo -e "\nWhat's your phone number?"
  read CUSTOMER_PHONE
  CUSTOMER_NAME=$($PSQL "SELECT name FROM customers WHERE phone = '$CUSTOMER_PHONE'")

  # if customer doesn't exist
  if [[ -z $CUSTOMER_NAME ]]
  then
    # get new customer name
    echo -e "\nI don't have a record for that phone number, what's your name?"
    read CUSTOMER_NAME

    # insert new customer
    INSERT_CUSTOMER_RESULT=$($PSQL "INSERT INTO customers(name, phone) VALUES('$CUSTOMER_NAME', '$CUSTOMER_PHONE')") 
  fi

  SERVICE_NAME=$($PSQL "SELECT name FROM services WHERE service_id = '$SERVICE_ID_SELECTED'")
  CUSTOMER_ID=$($PSQL "SELECT customer_id FROM customers WHERE phone='$CUSTOMER_PHONE'")
  CUSTOMER_NAME=$($PSQL "SELECT name FROM customers WHERE phone = '$CUSTOMER_PHONE'")
  echo -e "\nWhat time would you like your $SERVICE_NAME, $CUSTOMER_NAME?"
  read SERVICE_TIME

  # insert appoinment
  INSERT_APPOINTMENT_RESULT=$($PSQL "INSERT INTO appointments(customer_id, service_id, time) VALUES($CUSTOMER_ID, $SERVICE_ID_SELECTED, '$SERVICE_TIME')")
  if [[ $INSERT_APPOINTMENT_RESULT == "INSERT 0 1" ]]
  then
    # send to main menu
    echo -e "\nI have put you down for a $SERVICE_NAME at $SERVICE_TIME, $(echo $CUSTOMER_NAME | sed -r 's/^ *| *$//g').\n"
  fi 

}
    
MAIN_MENU

```

Execute that Bash file: `./salon.sh`.
This is the result:

```
$ ./salon.sh 

~~~~~ MY SALON ~~~~~

Welcome to My Salon, how can I help you?

1) cut
2) color
3) perm
4) style
5) trim
10

I could not find that service. What would you like today?
1) cut
2) color
3) perm
4) style
5) trim
sd

I could not find that service. What would you like today?
1) cut
2) color
3) perm
4) style
5) trim
f

I could not find that service. What would you like today?
1) cut
2) color
3) perm
4) style
5) trim
1

What's your phone number?
123

I don't have a record for that phone number, what's your name?
Sang

What time would you like your cut, Sang?
10am

I have put you down for a cut at 10am, Sang.

$ ./salon.sh 

~~~~~ MY SALON ~~~~~

Welcome to My Salon, how can I help you?

1) cut
2) color
3) perm
4) style
5) trim
2

What's your phone number?
123

What time would you like your color, Sang?
11am

I have put you down for a color at 11am, Sang.

$ ./salon.sh 

~~~~~ MY SALON ~~~~~

Welcome to My Salon, how can I help you?

1) cut
2) color
3) perm
4) style
5) trim
4

What's your phone number?
456

I don't have a record for that phone number, what's your name?
An

What time would you like your style, An?
8pm

I have put you down for a style at 8pm, An.
```

```
salon=# select * from services;
 service_id | name  
------------+-------
          1 | cut
          2 | color
          3 | perm
          4 | style
          5 | trim
(5 rows)

salon=# select * from customers;
 customer_id | name | phone 
-------------+------+-------
           1 | Sang | 123
           2 | An   | 456
(2 rows)

salon=# select * from appointments;
 appointment_id | customer_id | service_id | time 
----------------+-------------+------------+------
              1 |           1 |          1 | 10am
              2 |           1 |          2 | 11am
              3 |           2 |          4 | 8pm
(3 rows)

salon=# 

```

## Step 4: Dump DB into \<file>.sql

When completed, pls enter in the terminal to dump the database into a salon.sql file. It will save all the commands needed to rebuild it. Take a quick look at the file when you are done. The file will be located where the command was entered.  

```
pg_dump --clean --create --inserts --username=root -h localhost salon > salon.sql
```  

You can rebuild the database by entering in a terminal where the .sql file is.  

```
psql -h <hostname> -p <port> -U <username> -d <database> < <file>.sql
```  

Exp: `psql -h localhost -p 5432 -U root -d salon < salon.sql`