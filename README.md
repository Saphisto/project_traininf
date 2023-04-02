**this project will do the following:**  

1. download this git repository with dockerfile, requirments file and a flask application.  

2. will build a docker container, running the flask application and perform a test to check if its running.  

3. the test results will be saved to a file named report.csv and will be uploaded to S3.  

4. if the test was successful, the user will be given a choice on which machine to deploy the flask app.

**requirments:**  

EC2 instance for Jenkins controller.  

EC2 instance for development, where the Jenkins controller will test the application.  

EC2 instance for production where Jenkins Agent number 1 is.  

EC2 instance for production where Jenkins Agent number 2 is.  


installed Jenkins plugins.
1) 
