# How I set up ETL process and automated reporting in my company (using Python) - part 1

When I started my second data analyst position, I was challenged by a task I have never done before - creating a dashboard 
in Google Data Studio and make sure, it is all the time up-to-date. Yeah, and one more thing - the data are in three different sources, 
so it would be nice to build a warehouse first, that would be again, all the time up-to-date :)

Once I have learnt more about Airflow I set myself one more goal - reports written in Jupyter notebooks, automatically sent by email on regular basis, but I will save this for later.

I had to google literally everything, so I decided to write this article with all the steps I had to take in hope, somebody will benefit from it. All the coding part is done in Python.


### In this part of the article:


__1. Pulling data from the Google sheets__  
__2. Pulling data from the PostgreSQL database__  
__3. Pulling data from the Airtable__  
__4. Protecting credentials before pushing code to gitlab__  
__5. Writing own package of functions__  
__6. Creating a new PostgreSQL database with PgAdmin4__  
__7. Loading transformed data into the database (warehouse)__  
__8. Putting it all together (creating update_warehouse function)__
  
  
  
## 1. Pulling data from the Google sheets

First we need to create OAuth2 credentials.

1. Navigate in your browser to [Google APIs Console](https://console.developers.google.com/apis/dashboard?project=testdashboard&angularJsUrl=)  
2. Click on "TestDashboard" next to Google APIs logo, which will open dialog with all the projects you have on your account. Click on "New Project" on top right corner.  
![1_create_new project](https://user-images.githubusercontent.com/31499140/65150543-a39a7f80-da24-11e9-9e96-c2b93e38fc35.JPG)

3. Give your project a name and click "Create".
![2_Name_the_project](https://user-images.githubusercontent.com/31499140/65150775-1572c900-da25-11e9-97cc-2cee130cbd0c.JPG)

4. Click on "TestDasboard" again and select your project from the list.  
![3_Open_library](https://user-images.githubusercontent.com/31499140/65151341-3556bc80-da26-11e9-8039-5a447e53b35a.JPG)

5. Click on "ENABLE APIS AND SERVICES"
![4 Open_library_2](https://user-images.githubusercontent.com/31499140/65151501-85358380-da26-11e9-92d9-ea0eac94e351.JPG)

6. Search for and click on "Google Drive API"
![5 Search_GoogleDriveAPI](https://user-images.githubusercontent.com/31499140/65151685-e0677600-da26-11e9-92f4-c3e84217c49d.JPG)

7. Click "Enable" button.
![6 Enable](https://user-images.githubusercontent.com/31499140/65151890-4227e000-da27-11e9-8549-a846b0c0c473.JPG)

8. Now we need to create the credentials for a web server to access application data. Click on "Create credentials".
![7 CreateCredentials](https://user-images.githubusercontent.com/31499140/65225646-cf217680-dac5-11e9-98c4-4b683654b786.JPG)

9. Fill in the form accordingly and press the button below.
![7 CreateCredentials2](https://user-images.githubusercontent.com/31499140/65226101-946c0e00-dac6-11e9-9ee3-85fd895ed1f1.JPG)

10. Fill in your service account name and give a role project editor. Double check the key type is JSON and click "Continue".
![9 CreateCredentials3](https://user-images.githubusercontent.com/31499140/65226498-3ee43100-dac7-11e9-93ee-906235d15137.JPG)

11. Save the JSON file into your code folder and rename it to "client_secret.json"

12. Then go to the library again and enable "Google sheets API".

13. Open the file with a text editor and copy the client_email.
![10 client_secrets](https://user-images.githubusercontent.com/31499140/65227156-730c2180-dac8-11e9-8675-f035b7200f31.JPG)

14. Now we need to grand access to this email from the google sheet. So open the google sheet with the data in your browser, click 
on Share button in top right corner, add the email adress from your client_secret.json file and click "send".
![11 client_secret2](https://user-images.githubusercontent.com/31499140/65228110-2fb2b280-daca-11e9-953a-33ee55fa8026.JPG)


Once this is done, we are ready to pull the data to Python into a dataframe, so it is easy to work with and transform, if needed.

Before we start just make sure you have two packages installed:

oauth2client – to authorize with the Google Drive API using OAuth 2.0
gspread – to interact with Google Spreadsheets

install them with:  
```python
pip install gspread oauth2client
```

Once you have the packages installed you can pull the data into Python and get them in properly formated dataframe.
Set your working directory to the folder where you have the .json file or adjust the file path when setting credentials variable accordingly.

```python
import pandas as pd
import gspread
from oauth2client.service_account import ServiceAccountCredentials

# use credentails to create a client to interact with the Google Drive API
scope = ['https://spreadsheets.google.com/feeds','https://www.googleapis.com/auth/drive']
credentials = ServiceAccountCredentials.from_json_keyfile_name('client_secret.json', scope)
client = gspread.authorize(credentials)

# Find a workbook by name and open the sheet you need
# Make sure you don't misspell the name of the workbook and sheet.
sheet = client.open("Example").sheet1
data = pd.DataFrame(sheet.get_all_records())
```

## 2. Pulling data from the PostgreSQL database

I used psycopg2 package to create a connection to our database, so I will show you how to install that one. Furthemore you will need host, database, user and password. My IT colleague provided me with these, I believe it will be your scenario as well.

But be carefull here.  
We don't want to hard code the credentials into the code. Instead we will set the environment variables
with these values and use them in our code instead. This means that anybody who will want to be able to rerun your code will have to make sure to have the variables properly set before running the script. If you don't know how to set the environment variables,  go through the part 4. Protecting credentials before pushing code to gitlab. Since the ultimate goal is to make the script run automatically on Airflow server, I will show you later on, how to ensure that Airflow has also these environment variables set.

So first, installing the package. I ran into some errors, once I got to the point when I was installing the package within a docker and running the script on the Airflow server, but everything worked when I installed the package this way:

```python
pip install --no-binary :all: psycopg2
```

Once this is done, we are ready to pull the data.
```python
import os
import pandas as pd
import psycopg2

# now getting the credentials from the environment variables
host = os.environ['HOST_DATABASE']
database = os.environ['NAME_DATABASE']
user = os.environ['USER_DATABASE']
password = os.environ['PASSWORD_DATABASE']

conn = psycopg2.connect(host = host,database = database, user = user, password = password)

# creates a new cursor used to execute SELECT statements
cur = conn.cursor()

# use regular SELECT statement to pull the data you want
postgreSQL_select_Query = """
                          SELECT *
                          from orders_items
                        """
# quering the data
cur.execute(postgreSQL_select_Query)
data = cur.fetchall() 

# puting data into a dataframe
orders_items = pd.DataFrame.from_records(data, columns = ['date', 'customer', 'product', 'quantity'])

# closing the connection
cur.close()
conn.close()
```

If you are just starting with the SQL or you are writing a complicated statement, try it rather out somewhere else first (like PgAdmin), because python will not tell you where is an error in it, at least that was my case, it just froze.

And that was it, the only pain I had with this was putting data into a data frame, were I had to specify all the columns, so if you are pulling table with 20 columns it can get pretty tedious. If you know how to get the column names from the statement automatically, so I don't have to write them down, please let me know, thank you :)

## 3. Pulling data from the Airtable

To be able to pull data from Airtable you need to find the base key and api key. You will find out how to do it, for example
[in this article.](https://medium.com/row-and-table/an-basic-intro-to-the-airtable-api-9ef978bb0729)

Then we can move into the coding part. Just make sure you install airtable python wrapper.

```python
pip install airtable-python-wrapper
```

And now we are ready to pull the table, again into properly formated data frame. 
Again we are going to use environment variables, instead of hard coding the credentials into the script.

Since Airtable is a database, where multiple tables are connected, I had to deal with fixing the relations. Let's say we have three tables: Customers, Products and Orders. If you download Orders table, in column with product name you will end up with id of the product instead of its name, and the same for customer name column. Therefore I made sure, that I kept, not just the fields of the table I am downloading but also id of each row, so I could merge the tables later on correctly.

```python
import os
import pandas as pd
from airtable import Airtable

# pay attention to how I am using os.environ to call the environment variable from my computer, instead 
# of hard coding the value
airtable = Airtable(base_key = os.environ['AIRTABLE_BASE_KEY'], api_key=os.environ['AIRTABLE_API_KEY'], 
                               table_name = 'your_table_name')
airtable = airtable.get_all() 
airtable_df = pd.DataFrame.from_records(airtable)
# getting the fields with the data of the table
airtable = pd.DataFrame.from_records(airtable_df['fields'])

# adding column with the ID of each row, so the relations can be fixed
airtable['ID'] = airtable_df['id']
string = 'ID'
airtable.rename(columns={'ID': 'your_table_name + string}, inplace = True)
```

As you can see, I quickly ran into lot of data wrangling issues, which could get super messy and extend the main code, which wasn't necessary. Instead I took advice from my boss and wrote my own package with my own functions, that helped me to preprocess data from the airtable. I will show you how to do that, but let's take a look at how we set those environment variables first :)

## 4. Protecting credentials before pushing code to gitlab

As I wrote earlier, you don't want to keep your/company credentials in your code. Even though your code should be safe once you push it on for example gitlab into repository which only certain colleagues have access to, it is a rule in many companies not to push credentials nowhere in the repository. The way we protected our credentials was, that all the values I needed to make the code work, I saved in a group of environemt variables and sent the list of them to my colleagues over the email or Telegram. (Remember that Slack is not encrypted). 

### Setting the environment variable on Windows
1. In Search, search for and then select: System (Control Panel)
2. Click the Advanced system settings link.
3. Click Environment Variables. In the section System Variables, click on New.
4. In a dialog window specify name (for example BASE_API_KEY) and then specify the value, which is the corresponding string.
   Remember not to put in into quotes. It is also good practise to keep the name of the variables uppercase, for Mac users it is even
   necessery. Then you can just keep a .txt file with all the variables and their values, that you will send to your colleagues that 
   needs to run your scripts or are working on them with you.
5. Once you add all the environment variables you need close all the windows and restart your computer.

### Setting the environment variable on Mac
1. Open your command line and write:  open ~/.bash_profile
2. Once your .bash_profile file is open, write there down all the variables and values you need. Remember to use names with upper case. 
   For example BASE_API_KEY=keyd{skhsfe. Close the file.  
   Be careful! It happend with one of my values that contained special character that the letters after the character were not loaded.
   If you encounter the same thing you can put the value into single quotes, so you will end up with BASE_API_KEY='keyd{skhsfe'
3. In your command line write: source ~/.bash_profile
4. Reopen the terminal and try out that variables were set properly. Write for example echo $BASE_API_KEY and in the output you should
   get the value, you set for this variable. 

Thanks to source command, you don't need to restart your computer, the variables are already available. Just remember that they are 
available for the programs that starts from the terminal, for me it meant that even though I used to run the jupyter notebook and spyder from anaconda navigator, now I had to launch them directly from the command line.

From now on anytime you will need these values in your code, you will get them through os package.   
For example:
```python
import os

my_base_api_key = os.environ['BASE_API_KEY']
print(my_base_api_key)
```

## 5. Writing own package of functions

Before I built the warehouse I had to everytime download the tables from the Airtable and do the preprocessing, which was highly repetitive, extended the code and was uneffective. Solution to that was writing my own package of functions, that I just imported at the beginnig of each script. In my case it was mostly about preprocessing data from the Airtable but you can use it for anything.

To start to write your own package you need to write it in a seperate .py script. As an example I will demonstrate three functions.

Before I define them, I write a "help" part. It is very important that you write this part, since even though you are maybe the only one using these functions right now, it may not be this way always, or you might just forget how you designed them.


The first function is for unlisting the column. For some reason everytime I donwloaded the data, the date column was a list with one value, so I had to unlist it. On this function I also demonstrate how to incorporate a check that user is using the function properly and error message, in case condition is not met.  
We saw already the second function earlier, it is for downloading a table and preserving the ID of each row. Additionaly I am showing how to add some extra preprocessing steps for particular table and I am already using the first function for unlisting a date column.   
The third function is for fixing the relations, so if we continue in our example of three tables - Customers, Products and Orders,
thanks to this function in Orders table instead of customer ID and product ID we will get customer name and product name.

```python

"""
Example of usage:

from common import ReadAirtable_functions as ra

# Reading in the Tables
Customers = ra.ReadAirtable('Customers') # Only active customers are downloaded
Orders = ra.ReadAirtable('Orders') # Only orders with filled out date are downloaded, and date column is already unlisted.
Products = ra.ReadAirtable('Products')

# Fixing the relations (having an actual customer name instead of ID etc.)
Customers, Orders, Products = ra.FixRelations(Customers = Customers, Orders = Orders, Products = Products)

# In case you don't need to fix relations for all the tables
# I know that in this example we have only 3 tables, which are quite intuitive, but in reality you can have many more, so you need
  to document, which tables are connected with each other.
Customers, Orders, Products = ra.FixRelations(Customers = Customers, Orders = Orders)
del(Products)  # because these will be returned as None
# all possible pairs: Customers & Orders, Products & Orders
"""

# function for unlisting the column
def UnlistColumn(dataframe, column):
    if type(airtable[column][0]) == list:
        for index, row in dataframe.iterrows():
            dataframe[column][index]= dataframe[column][index][0]   
        return dataframe
    else:
        print('ERROR: Cannot apply UnlistColumn function on "' + column + '" column. It is not a list.')

# function for pulling the data and preprocessing
# I added here also some other steps like filtering rows that were not ready yet, unlisting date column etc., you can add if statement 
# as in example
def ReadAirtable( base_key = os.environ['AIRTABLE_BASE_KEY'], api_key = os.environ['AIRTABLE_API_KEY'], table_name):
    airtable = Airtable(base_key = base_key, api_key=api_key, table_name = table_name)
    airtable = airtable.get_all() 
    airtable_df = pd.DataFrame.from_records(airtable)
    airtable = pd.DataFrame.from_records(airtable_df['fields'])
    airtable['ID'] = airtable_df['id']
    string = 'ID'
    airtable.rename(columns={'ID': table_name + string}, inplace = True)
    
    if table_name == 'customers':
        airtable = airtable[airtable['active_customer'] == True].copy()
     
    if table_name == 'orders':
        airtable = airtable[airtable['Date'].notnull()].copy()
        airtable = UnlistColumn(airtable, 'Date') 
        
    
# function for getting actuall name instead of IDs.
# use Customers & Orders to get customer name into Orders table through customer ID
# use Products & Orders to get product name into Orduers through Product ID
# use all three of them, to fix them all
def FixRelations(Customers = None, Orders = None, Products = None):
    # Getting customer name into Orders table through customer ID
    if (Customers is not None) and (Orders is not None):
        Orders = (pd.merge(Orders, Customers[['Name', 'CustomersID']], left_on = 'Customer', right_on = 'CustomersID', 
                     how = 'left').drop('CustomersID', axis = 1))
        # Dropping Customer column with the IDs
        Orders.drop('Customer', axis = 1, inplace = True)
        # Renaming Name column to Customer
        Orders.rename(columns={'Name': 'Customer'}, inplace = True)
    
    # Getting product name into Orders through product ID
    if (Orders is not None) and (Products is not None):
        Orders = (pd.merge(Orders, Products[['Name', 'ProductsID']], left_on = 'Product', right_on = 'ProductsID', 
                     how = 'left').drop('ProductsID', axis = 1))
        # Dropping Product column with the IDs
        Orders.drop('Product', axis = 1, inplace = True)
        # Renaming Name column to Product
        Orders.rename(columns={'Name': 'Product'}, inplace = True)
```
 
Now save the script. In this example I named it ReadAirtable_functions.py and saved in a "common" folder in my working directory.
So next time I will be writing a report were I will need this data I will just do the import and use the functions:

```python
import os
import ReadAirtable_functions as ra

help(ra)
 
Customers = ra.ReadAirtable('Customers')
Orders = ra.ReadAirtable('Orders')
Products = ra.ReadAirtable('Products')

# Fixing the relations (having an actual customer name instead of ID etc.)
Customers, Orders, Products = ra.FixRelations(Customers = Customers, Orders = Orders, Products = Products)
```

So I didn't have to define the functions again, nor use huge amount of space to preprocess the data. Notice, that environment variables are specified directly in a script with the functions. We had only one Airtable database, so this way it was more convinient. If you are working with multiple Airtable databases you will have to adjust the function, so that airtable base key and airtable api key is specified when calling the function.


## 6. Creating a new PostgreSQL database with PgAdmin4

One of my tasks was to create a warehouse where data from all the sources would be stored. Additionaly there are extra tables with additional calculations, so all the dashboards in Google data studio and reports in jupyter notebook pulls data from there.  
I was creating a warehouse on an existing server, so the host, user, password was provided, I only add name for the new database.
First of all you need to design it. I used [drawio.](https://chrome.google.com/webstore/detail/drawio-desktop/pebppomjfocnoigkeepgbmcifnnlndla?hl=en-GB) to do that.    
The idea behind the warehouse is to store data in a simple way, so it is not necessary to use complex select statements where analyzing the data. Therefore you might want to go for dimensional database model instead of relational one. Probably you will be adding new tables as you will be doing more calculations on your data, but that is ok. Just keep updating the documentation accordingly, it will help a lot when you will be explaining your colleagues how to look for what they need in the warehouse.
One way or the other, somebody from the IT department will create server for you. Then you can use for example PgAdmin to connect to it. 

1. Open PgAdmin, right click on "Servers", then "Create" and "Server" wich will open a dialog window.
![1 connectServer](https://user-images.githubusercontent.com/31499140/65269631-e1c59b00-db19-11e9-9379-a4728d72f509.JPG)


2. On the General tab fill in the name, on the Connection tab fill in the host, username and password - all this information should   
   provide your IT department. Then click save.

3. Then on the left a new server name will appear. Click on "plus" next to it and right click on databases, then select "create" and "database". 
![2 createDatabase](https://user-images.githubusercontent.com/31499140/65269873-513b8a80-db1a-11e9-95a0-9b16d40340ca.JPG)

4. All you need to fill out is Database field, which will be the name of your database. I used "warehouse" in my example. Click save.
![3 createDatabase2](https://user-images.githubusercontent.com/31499140/65270306-2d2c7900-db1b-11e9-8fb0-56e22159b4ff.JPG)

Now you are ready to save the data into your database though Python.

## 7. Loading transformed data into the database (warehouse)

I am saving data into the PostgresSQL database through sqlalchemy package. There is no need for preprocessing the data once they are in a dataframe, just keep in mind that table and column names should be lowercase without space, for example customers_orders.

So let's say we want to save our three dataframes from the example: Orders, Products and Customers.

First package installation.
```python
pip install sqlalchemy
```

Make sure to create a new environment variable for your warehouse for create_engine function. The format is like this:
WAREHOUSE_ENGINE = postgresql+psycopg2://username:password@host:port/database_name

And now you need to add this into your script:
```python
from sqlalchemy import create_engine 

# creating the connection
# again, we are using environment variable instead of hardcoding the link.
engine_link = os.environ['WAREHOUSE_ENGINE']
engine = create_engine(engine_link)

# This will create new tables in your database, and replace the existing ones.
# use if_exists='append' to just add new rows
# notice that table names are lowercase now, to avoid problems when saving into the warehouse
Customers.to_sql('customers', engine, if_exists='replace',index=False)  
Products.to_sql('products', engine, if_exists='replace',index=False) 
Orders.to_sql('orders', engine, if_exists='replace',index=False) 

# closing the connection
engine.dispose()
```

That's it. All you need to do now is to right click on your database in PgAdmin, refresh it and all your tables will appear there, so you can double check, that all the data were saved properly.

## 8. Putting it all together (creating update_warehouse function)

With this knowledge you are ready to put together the main script with function that will do all the work. It is important to create a function with all the steps of ETL process in case you will want to run it automatically with Airflow.

So in case you have an extra script with your own functions for preprocessing the data your main script can look for example like this:

```python
import os
import pandas as pd
import gspread
from oauth2client.service_account import ServiceAccountCredentials
import psycopg2
from sqlalchemy import create_engine
from common import ReadAirtable_functions as ra # make sure to have proper working directory set

# defining some additional function that creates a new dataframe for example, which will be called from within the main function
def SomeAdditionalFunction(dataframe1, dataframe2)
    # extra code to transform the data
return dataframe3

# defining the main function, that does the whole ETL process
def update_warehouse():

  '''Pulling Google Sheet data'''
  # use credentails to create a client to interact with the Google Drive API
  scope = ['https://spreadsheets.google.com/feeds','https://www.googleapis.com/auth/drive']
  # make sure to set the proper path to the .json file, if it is not your working directory
  credentials = ServiceAccountCredentials.from_json_keyfile_name('client_secret.json', scope)
  client = gspread.authorize(credentials)

  # Find a workbook by name and open the sheet you need
  # Make sure you don't misspell the name of the workbook and sheet.
  sheet = client.open("Example").sheet1
  invoices = pd.DataFrame(sheet.get_all_records())

  '''Pulling Airtable data'''  
  Customers = ra.ReadAirtable('Customers')
  Orders = ra.ReadAirtable('Orders')
  Products = ra.ReadAirtable('Products')

  # Fixing the relations (having an actual customer name instead of ID etc.)
  Customers, Orders, Products = ra.FixRelations(Customers = Customers, Orders = Orders, Products = Products)
  
  '''Pulling PostgreSQL data'''
  # getting the credentials from the environment variables
  host = os.environ['HOST_DATABASE']
  database = os.environ['NAME_DATABASE']
  user = os.environ['USER_DATABASE']
  password = os.environ['PASSWORD_DATABASE']

  conn = psycopg2.connect(host = host, database = database, user = user, password = password)

  # creates a new cursor used to execute SELECT statements
  cur = conn.cursor()

  # use regular SELECT statement to pull the data you want
  postgreSQL_select_Query = """
                            SELECT *
                            from orders_items
                          """
  # quering the data
  cur.execute(postgreSQL_select_Query)
  data = cur.fetchall() 

  # puting data into a dataframe
  orders_items = pd.DataFrame.from_records(data, columns = ['date', 'customer', 'product', 'quantity'])

  # closing the connection
  cur.close()
  conn.close()
  
  # creating table with extra calculations
  customers_history = SomeAdditionalFunction(customers, orders_items)
  
  '''Saving the data into the warehouse'''
  # creating the connection
  engine_link = os.environ['WAREHOUSE_ENGINE']
  engine = create_engine(engine_link)

  # notice that table names are lowercase now, to avoid problems when saving into the warehouse
  Customers.to_sql('customers', engine, if_exists='replace',index=False)  
  Products.to_sql('products', engine, if_exists='replace',index=False) 
  Orders.to_sql('orders', engine, if_exists='replace',index=False) 
  orders_items.to_sql('orders_items', engine, if_exists='replace',index=False) 
  customers_history.to_sql('customers_history', engine, if_exists='replace',index=False) 
  

  # closing the connection
  engine.dispose()
  
  return 'Warehouse has been updated!'
  
# commented out call of the function, for easy run if needed
# update_warehouse()
```

Congratulations! You have just set up your first ETL process. Your secrets are safe, so you can push it in your teams gitlab. We pulled data from the Google sheets, Airtable and PostgreSQL database, we used functions from our own package, transformed data with extra function and saved it into the database you designed yourself!

In the next part of the article we will see how to pull data from your warehouse into Google Data Studio, set up the Airflow server in a docker so the IT department can right away put it on a production server and define tasks we want to run, while one of it will be jupyter notebook report sent on regular basis by email to your colleagues!

### References:
https://medium.com/row-and-table/an-basic-intro-to-the-airtable-api-9ef978bb0729
