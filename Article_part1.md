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
  
  
  
## 1. Pulling data from the Google sheets

First we need to create OAuth2 credentials.

1. Navigate in your browser to [Google APIs Console](https://console.developers.google.com/apis/dashboard?project=testdashboard&angularJsUrl=)  
2. Click on "TestDashboard" next to Google APIs logo, which will open dialog with all the project. Click on "New Project" on top right corner.
![1_create_new project](https://user-images.githubusercontent.com/31499140/65150543-a39a7f80-da24-11e9-9e96-c2b93e38fc35.JPG)

3. Give your project a name and click "Create".
![2_Name_the_project](https://user-images.githubusercontent.com/31499140/65150775-1572c900-da25-11e9-97cc-2cee130cbd0c.JPG)

4. Click on "TestDasboard" again and select your project from the list
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
host = os.environ['HOST_WAREHOUSE_DATABASE']
database = os.environ['NAME_WAREHOUSE_DATABASE']
user = os.environ['USER_WAREHOUSE_DATABASE']
password = os.environ['PASSWORD_WAREHOUSE_DATABASE']

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

If you are just starting with the SQL or you are writing a complicated statement, try it rather out somewhere else first (like PgAdmin), because python will not tell you where is an error in it, at least that was my case, it just kind of froze.

And that was it, the only pain I had with this was putting data into a data frame, were I had to specify all the columns, so if you are pulling table with 20 columns it can get pretty tedious. If you know how to get the column names from the statement automatically, so I don't have to write them down, please let me know, thank you :)

## 3. Pulling data from the Airtable

To be able to pull data from Airtable you need to find the base key and api key. You will find out how to do it, for example
[in this article](https://medium.com/row-and-table/an-basic-intro-to-the-airtable-api-9ef978bb0729)

Then we can move into the coding part. Just make sure you install airtable python wrapper.

```python
pip install airtable-python-wrapper
```

And now we are ready to pull the table, again into properly formated data frame. 
Again we are going to use environment variables, instead of hard coding the credentials into the script.

Since Airtable is a database, where multiple tables are connected, I had to deal with fixing the relations. Let's say we have three tables: Customers, Products and Orders. If you download Orders table, in column with Product name you will end up with id of the product instead of its name, and the same for customer name column. Therefore I made sure, that I kept, not just the fields of the table I am downloading but also id of each row, so I could merge the tables later on correctly.

```python
import os
import pandas as pd
from airtable import Airtable

# pay attention to how I am using os.environ to call the environment variable from my computer, instead of hard coding the value
airtable = Airtable(base_key = os.environ['AIRTABLE_BASE_KEY'], api_key=os.environ['AIRTABLE_API_KEY'], table_name = 'your_table_name')
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
