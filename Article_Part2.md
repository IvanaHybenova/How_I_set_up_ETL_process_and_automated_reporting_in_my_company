# How I set up ETL process and automated reporting in my company (using Python) - part 2  (in progress)

In first part of the article I showed you how I put data from all the sources together and a simple example of a script that saved
the gathered and transformed data to a new database (warehouse). The ultimate goal is to use Airflow to make the script run automatically
every night, so the data in the warehouse are still up to date.

But first, in this part of the article I will show you quickly, how to connect data to Google data studio, set up automatic refreshing 
of the browser and automatic refreshing of the data. This is especially handy if you are building dashboards for screens hanging on the 
walls in an office so it would be super annoying, if you would have to refresh them manualy every day.


### In this part of the article:


__1. Pulling data from postgreSQL into Google data studio__  
__2. Setting up refreshing data and browser automatically__  
__3. Writing your first DAG with tasks__  
__4. Pulling base docker image for Airflow server__  
__5. Fixing dockerfile__  
__6. Fixing airflow configuration file__  
__7. Fixing docker-compose.yml file__  
__8. Handing it all over to IT department__


## 1. Pulling data from postgreSQL into Google data studio

Google data studio is a free visualization tool and what I like about it most, is that you can connect it directly to your PostreSQL 
database. You can pull the whole table or write custom select statement, which allow you to combine data from multiple tables and add extra
calculations and aggregations on top of it. 

1. First step is to open your [Google data studio report overview](https://datastudio.google.com/u/0/navigation/reporting) and pick a new template.

2. Once you click on the template you want, you are redirected directly into edit mode of your report. Click on "CREATE NEW DATA SOURCE" in bottom right corner.  
![1 DataSource](https://user-images.githubusercontent.com/31499140/65310475-ac0ec980-db8e-11e9-9528-36e73b70d2e6.JPG)

3. Search for PostgreSQL connector and click select.  
![2 DataSource2](https://user-images.githubusercontent.com/31499140/65310597-f7c17300-db8e-11e9-97de-33061aba3353.JPG)

4. Fill in the host, port, database, username and password and click "AUTHENTICATE", these are the same values you used when you were   connecting to the database with Python.   
![3 DataSource3](https://user-images.githubusercontent.com/31499140/65310851-7cac8c80-db8f-11e9-96c3-32a84dc641b1.JPG)

5. Then either select the whole table, or write your custom query. Here is an example with finding top 10 customers based on number
of orders they made. Once you are happy with your query press connect, and all the columns you chose will appear in list of available
fields on the right. From here you just need to drag them into metrics and dimensions and repeat the process for every table and chart
you put into your dashboard.  
![4 DataSource4](https://user-images.githubusercontent.com/31499140/65311543-d5305980-db90-11e9-8b8b-1ded63f3a02e.JPG)
   
## 2. Setting up refreshing data and browser automatically

In my case I encountered with two problems - I needed the data to refresh automatically at least every hour, and I needed the browser 
to refresh everyday before 9AM, because we had a date filter on some of the dashboards, which gets updated with refreshing the page.
I am using Google Chrome so will provide you with solutions for this browser.

Let's deal with refreshing the data first. Google has a browser extension for this and it is called [Data studio Auto Refresh.](https://chrome.google.com/webstore/detail/data-studio-auto-refresh/inkgahcdacjcejipadnndepfllmbgoag)

After adding it to your browser navigate back to your report and refresh the browser. Then you will be able to left click on couple of blue arrows on top right corner and set automatic refresh of the data and pagination of the pages of your report. Don't miss the warning when downloading the extension - refreshing data from paid cloud database such as BigQuery cost money!  
![5 DataRefresh](https://user-images.githubusercontent.com/31499140/65311990-e29a1380-db91-11e9-9335-1c7ade9a11a6.JPG)


Now let's move on refreshing of the browser itself. For this you need [one of auto refreshers from google.](https://chrome.google.com/webstore/detail/auto-refresh/ifooldnmmcmlbdennkpdnlnbgbmfalko?hl=sk)
It is easy to set up, we only encountered one issue. After every reload of the page it left the full screen mode - the trick 
was to enter full screen mode via keybord shortcut instead of clicking on full screen button.

## 3. Writing your first DAG with tasks

If you are not familiar with Airflow at all, I recommend you to take a look at [this article.](http://michal.karzynski.pl/blog/2017/03/19/developing-workflows-with-apache-airflow/)  
You will understand the whole concept and it is also a tutorial about how to set up and run Airflow server on your local computer. Since it is not solution for production, I will not go over that, I will focus on running Airflow in a docker, because that is the way you need to build it, so the IT department will be able to put it into production.
But it is not a bad idea to play with Airflow server localy first, it is quick and easy to test your DAGs, when you don't have to run a docker container for it.

Now I want to show you example of a DAG. Actually two of them. The first one will run every night and will run our update_warehouse function. The second one will be scheduled to run once a month and will send a jupyter notebook report by email.

DAG file is actually a .py file, where you specify the arguments and tasks. I name the first one etl_DAG.py.

At the beginning of the script we will import the libraries and script with our update_warehouse function.

```python
# this is the script with your update_warehouse function from previous part of the article
from common import  warehouse_etl

from airflow import DAG
import datetime as dt
from airflow.operators.python_operator import PythonOperator
from airflow.utils.email import send_email
from airflow.operators.email_operator import EmailOperator
```
After imports I like to add notify_email function, which is a template for error messages I found in [this_article.](https://danvatterott.com/blog/2018/08/29/custom-email-alerts-in-airflow/)

```python
def notify_email(contextDict, **kwargs):
    """Send custom email alerts."""

    # email title.
    title = "Airflow alert: {task_name} Failed".format(**contextDict)
    
    # email contents
    body = """
    Hi Everyone, <br>
    <br>
    There's been an error in the {task_name} job.<br>
    <br>
    Forever yours,<br>
    Airflow bot <br>
    """.format(**contextDict)

    send_email('your_email@example.com', title, body) 
```

We will continue with setting up the default arguments for all the tasks in the DAG. Crucial part here is to understand how start_date works. It needs to be date from the past, because DAG run first after this given date. Another important argument is 'catchup'. From Airflow server you can run your DAGs also manually without messing with the set schedule. If I have DAG with start date 5 days ago and I would trigger it, the Airflow would run the DAG 5 times (once per each day, starting the start date until today, in case it is scheduled to run everyday).To avoid this behaviour we will later, on dag level, set catchup argument to False. Read more about this [here.](https://www.astronomer.io/blog/7-common-errors-to-check-when-debugging-airflow-dag/)

```python
default_args = {
        'owner': 'your_name',
        'start_date': dt.datetime(2019, 9, 16),  # remember it has to be date in the past
     #  'end_date': ,}  # we don't want end date in this case, so it is commented out
        'depends_on_past': False,
        'email': ['your_email@example.com'],
        'email_on_failure': False,  # we are setting this to False, because we want to receive our custom email instead
        'retries': 1,
        'retry_delay': dt.timedelta(minutes = 2)}
        
```
Here comes the body of the DAG. The most headache will come from schedule_interval argument, especially if you need to run the dag for example each 15. day of the month at 2AM. Remember, that Airflow is using UTC timezone. To get some idea see the [documentation](https://airflow.apache.org/scheduler.html) and this [stack overflow discussion](https://stackoverflow.com/questions/35668852/how-to-configure-airflow-dag-to-run-at-specific-time-on-daily-basis)

We will set this DAG with etl process every day.

```python
# DAG that will run etl process
dag = DAG(
        'etl',
        default_args = default_args,
        description = 'ETL process',
        # Continue to run DAG once per day
        schedule_interval = '@daily') # the DAG will run after midnight 
```

Now we will add the tasks. In our sipmle example we have only one task to run, so we will add one more, an email notification about successful completion of the update task sent to your email. Just to demonstrate how to set the dependencies and order of the tasks.

```python
# first comes PythonOperator, that runs the script with our function in python_callable argument
warehouse_operator = PythonOperator(task_id = 'warehouse_task',  # name is completely up to you
                                    python_callable = warehouse_etl.update_warehouse # make sure to set proper name of the script and the function
                                    , on_failure_callback=notify_email,  # here comes the trick with custom notification email
                                    dag = dag) 

success_notification_operator = EmailOperator (
        
                                                          dag=dag,
                                                          task_id="notification_email",
                                                          to=["your_email@example.com"],
                                                          subject="Airflow update",
                                                          html_content="<h3>Warehouse update was successful</h3>",
                                                          on_failure_callback=notify_email)

# setting up the order of the tasks
warehouse_operator >> success_notification_operator
```

I will name the second DAG monthly_reports_DAG.py and it will look very similar. We will just add two more libraries and we assume that in our common folder is alongside with the scripts also notebook folder with jupyter notebook named monthly_sales_report.ipynb. The exact folder structer will be presented later on, so you will get the whole picture, don't worry. The idea behind this, is to convert the notebook into an .html file, while skipping the code. So only the markdown sections and outputs will be visible for in the .html file, which makes it a great way of automated reporting.

Another thing I wanted to show you in this example is picking up the files in your docker. We will take advantage of AIRFLOW_HOME environment variable, because its value is a path. Thanks to that and Path function from pathlib package we can get to any file in the docker container we need. I will get back to this, once we will be talking about the folder structure, now just follow along, please.

```python
# importing the libraries
from airflow import DAG
import datetime as dt
from airflow.operators.python_operator import PythonOperator
from airflow.utils.email import send_email
from airflow.operators.email_operator import EmailOperator

# two extra libraries that will help us to locate the jupyter notebook we want to send
from pathlib import Path
import os

# and one more library, that will allow us to run the notebook and give us html file without code as an output
from airflow.operators.bash_operator import BashOperator

# the same default arguments
default_args = {
        'owner': 'your_name',
        'start_date': dt.datetime(2019, 9, 16),  # remember it has to be date in the past
     #  'end_date': ,}  # we don't want end date in this case, so it is commented out
        'depends_on_past': False,
        'email': ['your_email@example.com'],
        'email_on_failure': False,  # we are setting this to False, because we want to receive our custom email instead
        'retries': 1,
        'retry_delay': dt.timedelta(minutes = 2)}
        

# DAG that will send the report once a month
dag = DAG(
        'etl',
        default_args = default_args,
        description = 'Reporting',
        # Continue to run DAG once per month
        schedule_interval = '@monthly') # the DAG will run at the beginning of each month 

# Now we add an BashOperator 
monthly_report_operator = BashOperator(task_id='sales_monthly_report', 
                                       bash_command='jupyter nbconvert --execute --to html $AIRFLOW_HOME/dags/common/notebooks/monthly_sales_report.ipynb --no-input', 
                                                      on_failure_callback=notify_email,dag=dag)

# The EmailOperator will take the produced .html file and send it to given email adress/-es
monthly_report_email_operator = EmailOperator (
        
                                                          dag=dag,
                                                          task_id="send_sales_monthly_report",
                                                          to=["your_email@example.com", "your_colleague@example.com"],
                                                          subject="Sales report",
                                                          files = [Path(os.environ['AIRFLOW_HOME']+'/dags/common/notebooks/monthly_sales_report.html')],
                                                          html_content="<h3>Report sent by Airflow - download before opening!</h3>",
                                                          on_failure_callback=notify_email)

# now setting the order of the operators
monthly_report_operator >> monthly_report_email_operator
```

Wow, that was piece of work, right? :) But there is still long way ahead to make all this work. Let's start with the dockerfile.

## 4. Pulling base docker image for Airflow server

As a starting point I used this [Airflow tutorial.](https://gosmarten.com/airflow.html)
As the tutorial states, the first step is to clone the repository with the whole set up of dockerfile, airflow configuration file, compose.ymal file, etc. But I have also kept the whole folder structer as is in the repository. Once you are more comfortable and familiar with the configuration file, don't hasitate to come up with your own folder structure and adjust the files accordingly. 
We will do some little changes though.

```python
git clone https://github.com/gosmarten/airflow.git
```

The Dockerfile lives in docker/airflow/Dockerfile, but I had to move it to the root folder of the repository, so I ended up with slightly different structure. So, move the Dockerfile into root directory of the cloned repository, so it is in the same folder as docker folder and src folder.   
![1 Dockerfile_localization](https://user-images.githubusercontent.com/31499140/65350781-c96f8200-dbe6-11e9-9749-5039eb0d5bb8.JPG)

Next change is going to be made in docker/airflow/config folder. Here is located airflow.cfg file. Rename this file to airflow.cfg.template and copy here also client_secret.json file (if you need it) and rename it to client_secret.json.template.

So in your docker/airflow/config folder you should have these two:  
![2 config folder](https://user-images.githubusercontent.com/31499140/65351504-93cb9880-dbe8-11e9-8cfa-c260931265b2.JPG)

Be careful! Before you push this stage to gitlab make sure to do one change in client_secret.json.template file. Open it in your text editor and replace the value of private_key with some environment variable, for example $PRIVATE_KEY. Just do remember it.  
![3  client_secret_template](https://user-images.githubusercontent.com/31499140/65351704-0d638680-dbe9-11e9-87bf-88e40fe83e8a.JPG)

I know that this make no sense right now, but I will explain little later, how this trick is going to protect your secret credentials :)

And don't forget to open requirements.txt file located in docker/airflow/requirements.txt and replace it with packages needed for your project!

At this point you can add your DAG files and all the other scripts and notebooks. So continuing in our example:  
1. the etl_DAG.py and monthly_reports_DAG.py will be located in src/dags,   
2. the warehouse_etl.py will be located in src/dags/common,
3. the monthly_sales_report.ipynb will be located in src/dags/common/notebooks.  

Now let's adjust the dockerfile according to our needs.

## 5. Fixing dockerfile

The first change is line 44, I like to add vim, to be able easily check log files in case of error and we will need also gettext-base,
so add these two after locales:  
![4 fixingDocker1](https://user-images.githubusercontent.com/31499140/65352271-86afa900-dbea-11e9-9d0a-cfaf46675ecc.JPG)

Right before "# Adding installation of custom requirements" we will add section with environment variables. Some of the variables are not that secret, but in case they will change it is easier to change them in Dockerfile only, then looking for them in all the scripts you will ever write. So the environment variables that are not that secret will be defined here, and those that are secret, like passwords etc. will be written in docker-compose.ymal file (this docker-compose.ymal file will never be pushed to gitlab after we fix it).  
I like to keep noted in dockerfile list of environment variables that are specified in docker-compose.ymal file, so the next section looks like this for our example of warehouse_update function from previous part of the article:

```
# Company variables

ENV HOST_DATABASE="your_host"
ENV NAME_DATABASE="database_name"
ENV USER_DATABASE="user_name"

# In docker-compose.ymal file:
# ENV WAREHOUSE_ENGINE
# ENV PASSWORD_DATABASE
# ENV AIRTABLE_BASE_KEY
# ENV AIRTABLE_API_KEY
# ENV WAREHOUSE_ENGINE

ENV AIRFLOW_HOME=${AIRFLOW_HOME}
```

Rest of the Dockerfile is mostly about copying all our files into docker container. If you will be changeing structer of the folder, make sure to adjust all COPY commands accordingly.   

```
# Adding installation of custom requirements
COPY docker/airflow/requirements.txt ${AIRFLOW_HOME}/requirements.txt
RUN pip install -r ${AIRFLOW_HOME}/requirements.txt
RUN pip install --no-binary :all: psycopg2

COPY docker/airflow/script/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh        # to make sure the entrypoint.sh file is executable

COPY docker/airflow/config/airflow.cfg.template ${AIRFLOW_HOME}/airflow.cfg.template

COPY docker/airflow/config/client_secret.json.template ${AIRFLOW_HOME}/client_secret.json.template

COPY src/dags ${AIRFLOW_HOME}/dags

RUN chown -R airflow: ${AIRFLOW_HOME}


EXPOSE 8080 5555 8793

USER airflow

WORKDIR ${AIRFLOW_HOME}
RUN export PYTHONPATH=${PYTHONPATH}:${AIRFLOW_HOME}
ENTRYPOINT ["/entrypoint.sh"]
```

Notice that we have set the working directory to ${AIRFLOW_HOME}, where we have also coppied the dags folder. If you recall, the jupyter notebook is located in /dags/common/notebooks folder, that is why the path for it in the script was $AIRFLOW_HOME/dags/common/notebooks/monthly_sales_report.ipynb.  
If you will have different structure from this one, adjust the script accordingly.  
The Dockerfile is ready, now we can move on to airflow.cfg.template file.

## 6. Fixing airflow configuration file

Open your airflow.cfg.template file in a text editor, we are going to fix some issues here.  
In our set up, the IT department decided to rebuild the docker image everytime there are some changes to the dags. For this case it is necessary to make sure, that everytime the Airflow server starts to run, all the DAGs are automatically ON. By default are DAG paused, so the first change is made in row number 52.  
![5 FixingConfigurationFile1](https://user-images.githubusercontent.com/31499140/65355451-3e948480-dbf2-11e9-9677-ba7fe4da5ec4.JPG)

Now we need to set up the [smtp] section, so the Airflow server will be able to send emails.

```
[smtp]
# If you want airflow to send emails on retries, failure, and you want to use
# the airflow.utils.email.send_email_smtp function, you have to configure an
# smtp server here
smtp_host = smtp.gmail.com
smtp_starttls = True
smtp_ssl = False
# Uncomment and set the user/pass settings if you want to use SMTP AUTH
smtp_user = your_airflow_email@example.com
smtp_password = $SMTP_PASSWORD   
# it was 25 originally
smtp_port = 587   
smtp_mail_from = your_airflow_email@example.com
```

The your_airflow_email@example.com is an email account you need to create, for example on gmail.com. $SMTP_PASSWORD will be set in docker-compose.ymal file as it is a secret password to that email account.

There are many other things you can set in the configuration file and many of them are pretty intuitive, if you don't understand something, you will be for sure able to google it.

The only thing left now is fixing docker-compose.ymal file, so all the secret environment variables will be properly set and safe.

## 7. Fixing docker-compose.ymal file



### References
http://michal.karzynski.pl/blog/2017/03/19/developing-workflows-with-apache-airflow/  
https://danvatterott.com/blog/2018/08/29/custom-email-alerts-in-airflow/  
https://airflow.apache.org/scheduler.html  
https://www.astronomer.io/blog/7-common-errors-to-check-when-debugging-airflow-dag/  
https://gosmarten.com/airflow.html
