EDISS - A2:
Download node modules :
npm install
npm init -y

//Check JWT validation

.env for BFF on SSH: Different for all
DB_HOST=bookstore-db-dev.cluster-cdcmvzkmrbdr.us-east-1.rds.amazonaws.com
DB_USER=root
DB_PASSWORD=edispass
DB_NAME=bookstore
PORT=3000
URL_BASE_BACKEND_SERVICES_BOOK=http://internal-bookstore-dev-InternalALB-1000562200.us-east-1.elb.amazonaws.com:3000
URL_BASE_BACKEND_SERVICES_CUSTOMER=http://internal-bookstore-dev-InternalALB-1000562200.us-east-1.elb.amazonaws.com:3000

****** Use internal load balancer’s DNS

.env for book & customer-service on SSH:


DB_HOST=bookstore-db-dev.cluster-cdcmvzkmrbdr.us-east-1.rds.amazonaws.com
DB_USER=root
DB_PASSWORD=edispass
DB_NAME=bookstore
PORT=3000


## BUILD, TAG, PUSH Docker Image

##BUILD
docker buildx build --platform linux/amd64,linux/arm64 \
-t jigyasag/ediss-a2:book-service-v1 \
book-service --push

docker buildx build --platform linux/amd64,linux/arm64 \
  -t jigyasag/ediss-a2:customers-service-v1 \
  customer-service --push
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jigyasag/ediss-a2:web-bff-service-v1 \
  web-bff --push
docker buildx build --platform linux/amd64,linux/arm64 \
  -t jigyasag/ediss-a2:mobile-bff-service-v1 \
  mobile-bff --push

****Alternate -Use a script with Docker compose to automate Pulling images and docker run ******
##PULL all Images on the instances - here 4 instances

docker pull jigyasag/ediss-a2:books-service-v1
docker pull jigyasag/ediss-a2:customers-service-v1
docker pull jigyasag/ediss-a2:web-bff-service-v1
docker pull jigyasag/ediss-a2:mobile-bff-service-v1

********RUN******* RDS Writer endpoint

###### EC2BookstoreA
web-bff (port 80)

customer-service (port 3000)

####### EC2BookstoreB
web-bff (port 80)

book-service (port 3000)

######## EC2BookstoreC
mobile-bff (port 80)

book-service (port 3000)

######### EC2BookstoreD
mobile-bff (port 80)

customer-service (port 3000)

----------------------

docker run -d -p 3000:3000 \
  -e DB_HOST=bookstore-db-dev.cluster-cdcmvzkmrbdr.us-east-1.rds.amazonaws.com \
  -e DB_USER=root \
  -e DB_PASSWORD=edispass \
  -e DB_NAME=bookstore \
  -e PORT=3000 \
  jigyasag/ediss-a2:customers-service-v1

------------------------
docker run -d -p 3000:3000 \
  -e DB_HOST=bookstore-db-dev.cluster-cdcmvzkmrbdr.us-east-1.rds.amazonaws.com \
  -e DB_USER=root \
  -e DB_PASSWORD=edispass \
  -e DB_NAME=bookstore \
  -e PORT=3000 \
  jigyasag/ediss-a2:book-service-v1

------------------------

docker run -d -p 80:80 \
  -e DB_HOST=bookstore-db-dev.cluster-cdcmvzkmrbdr.us-east-1.rds.amazonaws.com \
  -e DB_USER=root \
  -e DB_PASSWORD=edispass \
  -e DB_NAME=bookstore \
  -e PORT=80 \
 -e URL_BASE_BACKEND_SERVICES_CUSTOMER=http://internal-bookstore-dev-InternalALB-1000562200.us-east-1.elb.amazonaws.com:3000 \
 -e URL_BASE_BACKEND_SERVICES_BOOK=http://internal-bookstore-dev-InternalALB-1000562200.us-east-1.elb.amazonaws.com:3000 \
  jigyasag/ediss-a2:web-bff-service-v1

------------------------

docker run -d -p 80:80 \
  -e DB_HOST=bookstore-db-dev.cluster-cdcmvzkmrbdr.us-east-1.rds.amazonaws.com \
  -e DB_USER=root \
  -e DB_PASSWORD=edispass \
  -e DB_NAME=bookstore \
  -e PORT=80 \
   -e URL_BASE_BACKEND_SERVICES_CUSTOMER=http://internal-bookstore-dev-InternalALB-1000562200.us-east-1.elb.amazonaws.com:3000 \
 -e URL_BASE_BACKEND_SERVICES_BOOK=http://internal-bookstore-dev-InternalALB-1000562200.us-east-1.elb.amazonaws.com:3000 \
  jigyasag/ediss-a2:mobile-bff-service-v1

**************

Alternate:

Docker compose:
B: 
services:
  web-bff:
    image: jigyasag/web-bff:latest
    container_name: web-bff
    restart: always
    ports:
      - "80:80"
    env_file:
      - .env

  customer-service:
    image: jigyasag/book-service:latest
    container_name: book-service
    restart: always
    ports:
      - "3000:3000"
    env_file:
      - .env

C:

services:
  web-bff:
    image: jigyasag/mobile-bff:latest
    container_name: mobile-bff
    restart: always
    ports:
      - "80:80"
    env_file:
      - .env

  customer-service:
    image: jigyasag/book-service:latest
    container_name: book-service
    restart: always
    ports:
      - "3000:3000"
    env_file:
      - .env

D: 

services:
  web-bff:
    image: jigyasag/mobile-bff:latest
    container_name: mobile-bff
    restart: always
    ports:
      - "80:80"
    env_file:
      - .env

  customer-service:
    image: jigyasag/customer-service:latest
    container_name: customer-service
    restart: always
    ports:
      - "3000:3000"
    env_file:
      - .env

*****************On any of the instances*********** RDS Writer endpoint

sQL: 
mysql -h bookstore-db-dev.cluster-cdcmvzkmrbdr.us-east-1.rds.amazonaws.com -u root -pedispass


SCHEMA: 
## Use Books not Book
## Use Customers not Customer
 CREATE DATABASE IF NOT EXISTS bookstore;
USE bookstore;
DROP TABLE Books;
DROP TABLE Customers;
CREATE TABLE IF NOT EXISTS Customers (
   id INT AUTO_INCREMENT PRIMARY KEY,
   userId VARCHAR(255) UNIQUE NOT NULL,
   name VARCHAR(100) NOT NULL,
   phone VARCHAR(20),
   address VARCHAR(255),
   address2 VARCHAR(255),
   city VARCHAR(100),
   state VARCHAR(50),
   zipcode VARCHAR(20)
);
CREATE TABLE IF NOT EXISTS Books (
   ISBN VARCHAR(20) PRIMARY KEY,
   title VARCHAR(255) NOT NULL,
   Author VARCHAR(100) NOT NULL,
   description TEXT NOT NULL,
   genre VARCHAR(100) NOT NULL,
   price DECIMAL(10,2) NOT NULL,
   quantity INT NOT NULL
);


*********BEFORE EVERY RUN*******
TRUNCATE TABLE Books;
TRUNCATE TABLE Customers;
 
 ***** For debugging****
 docker ps
 docker exec -it <container-name>
 docker logs <container name>
