# How I set up ETL process and automated reporting in my company (using Python) - part 2  (in progress)

In first part of the article I showed you how I put data from all the sources together and a simple example of a script that saved
the gathered and transformed data to a new database (warehouse). The ultimate goal is to use Airflow to make the script run automatically
every night, so the data in the warehouse are still up to date.

But first in this part of the article I will show you quickly, how to connect data to Google data studio, set up automatic refreshing 
of the browser and automatic refreshing of the data. This is especially handy if you are building dashboards for screens hanging on the 
walls in an office so it would be super annoying, if you would have to refresh them manualy everyday.


### In this part of the article:


__1. Pulling data from postgreSQL into Google data studio__  
__2. Setting up refreshing data and browser automatically__  
__3. Writing your first DAG with tasks__  
__4. Pulling base docker image for Airflow server__  
__5. Fixing dockerfile__  
__6. Fixing airflow configuration file__  
__7. Fixing .ymal file__  
__8. Handing it all over to IT department__


## 1. Pulling data from postgreSQL into Google data studio

Google data studio is a free visualization tool and what I like about it most, is that you can connect it directly to your PostreSQL 
database. You can pull the whole table or write custom select statement, which allow you combine data from multiple tables and add extra
calculations and aggregations on top of it. 

1. First step is to open you [Google data studio report overview](https://datastudio.google.com/u/0/navigation/reporting) and pick a new template.

2. Once you click on template you want, you are redirected directly into edit mode of your report. Click on "CREATE NEW DATA SOURCE" on bottom
right corner.
![1 DataSource](https://user-images.githubusercontent.com/31499140/65310475-ac0ec980-db8e-11e9-9528-36e73b70d2e6.JPG)

3. Search for PostgreSQL connector and click select.
![2 DataSource2](https://user-images.githubusercontent.com/31499140/65310597-f7c17300-db8e-11e9-97de-33061aba3353.JPG)

4. Fill the host, port, database, username and password and click "AUTHENTICATE", these are the same values you used when you were connecting
   to the database with Python.
   ![3 DataSource3](https://user-images.githubusercontent.com/31499140/65310851-7cac8c80-db8f-11e9-96c3-32a84dc641b1.JPG)

5. Then either select the whole table, or write your custom query. Here is an example with finding top 10 customers based on number
   of orders they made. Once you are happy with your query press connect, and all the columns you chose will appear in list of available
   field on the right. From here you just need to drag them into metrics and dimensions and repeat the process for every table and chart
   you put into your dashboard.
   ![4 DataSource4](https://user-images.githubusercontent.com/31499140/65311543-d5305980-db90-11e9-8b8b-1ded63f3a02e.JPG)
   
## 2. Setting up refreshing data and browser automatically

In my case I encountered with two problems - I needed the data to refresh automatically at least every hour, and I needed the browser 
to refresh everyday before 9AM, because we had a date filter on some of the dashboards, which gets updated with refreshing the page.
I am using Google Chrome so will provide you with solutions for this browser.

Let's deal with refreshing the data first. Google has a browser extension for this and it is called [Data studio Auto Refresh.](https://chrome.google.com/webstore/detail/data-studio-auto-refresh/inkgahcdacjcejipadnndepfllmbgoag)

When you navigate back to your report and refresh the browser you will be able to left click on couple of blue arrows on top right corner
and set automatic refresh of the data and paginating of the pages of your report. Don't miss the warning when downloading the extension;
refreshing data from paid cloud database such as BigQuery cost money!
![5 DataRefresh](https://user-images.githubusercontent.com/31499140/65311990-e29a1380-db91-11e9-9335-1c7ade9a11a6.JPG)


Now let's move on refreshing of the browser itself. For this you need [one of auto refreshers from google](https://chrome.google.com/webstore/detail/auto-refresh/ifooldnmmcmlbdennkpdnlnbgbmfalko?hl=sk)
It is easy straightforward to use, we only encountered one issue. After every reload of the page it left the full screen mode - the trick 
was to enter full screen mode via keybord shortcut instead of clicking on full screen button.

## 3. Writing your first DAG with tasks
