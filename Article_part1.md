# How I set up ETL process and automated reporting in my company - part 1

When I started my second data analyst position, I was challenged by a task I have never done before - creating a dashboard 
in Google Data Studio and make sure, it is all the time up-to-date. Yeah, and one more thing - the data are in three different sources, 
so it would be nice to build a warehouse first, that would be again, all the time up-to-date :)

Once I have learnt more about Airflow I set myself one more goal - reports written in Jupyter notebooks, automatically sent by email on regular basis, but I will save this for later.

I had to google literally everything, so I decided to write this article with all the steps I had to take in hope, somebody will benefit from it.


### In this part of the article:


__1. Pulling data from the Google sheets__  
__2. Pulling data from the PostgreSQL database__  
__3. Pulling data from the Airtable__  
__4. Protecting credentials before pushing code to gitlab__  
__5. Writing own package of functions__  
__6. Creating a new PostgreSQL database with PgAdmin4__  
__7. Loading transformed data into the database (warehouse)__  
  
  
  
## 1. Pulling data from the Google sheets

I decided to use [gspread](https://github.com/burnash/gspread) package for this, so make sure you install it first.

Now we need to create OAuth2 credentials.

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

