---
layout: post
title: Set Up Dual Databases in Rails
writer_photo_url: /images/team/deepak.jpeg
writer: Deepak Chauhan
---

We encountered a requirement to set up dual databases in Rails. Our app is hosted on Heroku, primarily using the Postgres database add-on. Additionally, we decided to create another database on an AWS RDS instance, allowing our Heroku app to point to these two databases.

### Creating an Amazon RDS Postgres Instance
1. Log in to AWS.
2. Search for RDS.
3. Create a database with the following configuration:
- Templates: Production
- Availability and Durability: Single DB Instance
- Instance Configuration: Burstable class, db.t4g.small - vCPU: 2, Memory: 2 GB, EBS Burst Bandwidth: 2085 Mbps
- Storage: 20 GB (Auto-scaling enabled)
- Automated Backup: Available (7-day default period)
- Estimated Monthly Cost: 26.49 USD
This is the basic configuration for our AWS RDS instance; customize it according to your needs.
![AWS RDS instance config](/assets/img/aws_rds_instance_config.png)

For connectivity and security group, open the VPC and security group to find one default VPC security group. Edit the inbound rule by deleting the existing rule and adding a new custom rule. Under the type dropdown, select PostgreSQL. For the source value, select "Anywhere IPv4," then save the rule.
![AWS VPC security group](/assets/img/vpc_security_group.png/)

### Testing Connection to AWS RDS Instance
1. Through DBeaver (GUI)
- Endpoint: wrep-db-instance.cepwonio1pq8976.us-east-1.rds.amazonaws.com (your instance endpoint)
- Port: 5432
- Master Username: your username
- Master Password: your password
Test the connection and then finish.

![Dbeaver connection](/assets/img/dbeaver_connection.png)

2. Through Terminal

```bash
psql -h wrep-db-instance.cepwonio1pq8.us-east-1.rds.amazonaws.com -p 5432 -U username -d postgres
```
3. After connecting, manually create a database:

```sql
CREATE DATABASE your_database_name;
```
For our case, we named it wrep:

```sql
CREATE DATABASE wrep;
```
Rails does most of the work for you, but we still need to modify our Rails app to accommodate the multiple database structure.

### Two-Tier Vs. Three-Tier Database Configuration
For multiple databases, write database configurations in the three-tier fashion in database.yml.

Example
For a single database, we write it in a two-tier fashion like below:

```yaml
production:
  <<: *default
  database: mymoc_production
  url: <%= ENV['DATABASE_URL'] %>
```
Whereas a three-tier will look like below:

```yaml
production:
  primary:
    adapter: postgresql
    url: <%= ENV['DATABASE_URL'] %>
    migrations_paths: db/migrate/primary
  wrep:
    adapter: postgresql
    url: <%= ENV['WREP_DATABASE_URL'] %>
    migrations_paths: db/migrate/wrep
```

When Rails finds this three-tier configuration in database.yml, it treats your Rails application as a multi-database application. Change the default two-tier configuration to a three-tier configuration in database.yml.

Final database.yml Configuration
```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  primary:
    <<: *default
    database: mymoc_development
    migrations_paths: db/migrate/primary
  wrep:
    <<: *default
    database: wrep_development
    migrations_paths: db/migrate/wrep

# Similar for test, staging, and production...
```

### Creating Models and Migrations
By default, models in a Rails application inherit from ```ApplicationRecord``` and connect to a single database. Now, we want some models to connect to the primary database and some to the wrep database.

```ruby
class Sample < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :wrep }
end
```
connects_to tells Rails that the Sample model will connect to the wrep database for both writing and reading purposes.

Create a base class for all sample models to avoid repeating this configuration:

```ruby
class SampleRecord < Sample
  self.table_name = "sample"
end
```

### Migration Folder Structure Change
Move existing migrations from db/migrate to db/migrate/primary, and create another folder db/migrate/wrep for wrep-related migrations:

```bash
mv ./db/migrate/* ./db/migrate/primary/
```
### Some Rules to Follow
- To generate a migration file for the WREP database in ```db/migrate/wrep```, use: ```rails g migration --database=wrep```.
- To migrate the primary database, run: ```rails db:migrate:primary```.
- To migrate the Rules Engine Database, run: ```rails db:migrate:wrep```.
- Running ```rails db:migrate``` will now execute migrations for both databases.

### Deploying Multi-Database App on Heroku
Update the production section of database.yml. Heroku relies on the ```DATABASE_URL``` environment variable. Set an environment variable like ```WREP_DATABASE_URL``` in this format for the wrep AWS RDS instance database:

```yaml
production:
  primary:
    adapter: postgresql
    url: <%= ENV['DATABASE_URL'] %>
    migrations_paths: db/migrate/primary
  wrep:
    adapter: postgresql
    url: <%= ENV['WREP_DATABASE_URL'] %>
    migrations_paths: db/migrate/wrep
```

### Authorizing Access to RDS Instance
The section on "Authorizing Access to RDS Instance" is crucial for securely connecting your Heroku application to an Amazon RDS instance, especially when dealing with sensitive data or ensuring secure and reliable service operations. 
### Granting Heroku Dynos Access to Your RDS Instance
Heroku applications run on dynos, which are isolated, virtualized Linux containers designed to execute code based on a user-specified command. When your application needs to interact with an external database hosted on Amazon RDS, you must explicitly grant access to these dynos for security reasons. By default, RDS instances are not publicly accessible, and access is tightly controlled through AWS security groups, which act as virtual firewalls.

Configuring a Heroku Ruby App to Use a Postgres RDS Instance
Download the Amazon RDS CA certificate:
```bash
curl https://s3.amazonaws.com/rds-downloads/rds-combined-ca-bundle.pem > ./config/amazon-rds-ca-cert.pem
```
you're ensuring that your application has the necessary certificate to establish a trusted connection to the RDS instance.

Add the certificate file to your app’s git repository and redeploy to Heroku.
Update the ```WREP_DATABASE_URL``` config var to include the sslca parameter:
```bash
heroku config:set DATABASE_URL="postgres://username:password@hostname/dbname?sslca=config/amazon-rds-ca-cert.pem" -a <app_id>
```
The ```sslca``` parameter in the database URL tells your application to use the specified CA certificate for SSL connection to the RDS instance. This ensures that your application connects securely using SSL and that the RDS instance's SSL certificate is validated against the CA certificate you've provided.

Now, our app is configured to use multiple databases on Heroku. Deploy and ```heroku run rails db:migrate -a app-name``` to set everything up, and you can start using multiple databases on Heroku with Rails.
