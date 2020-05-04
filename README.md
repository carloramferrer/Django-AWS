# Deploying your Django Application to AWS Elastic Beanstalk
These are detailed steps on how to deploy your Django application on AWS S3 File Storage and Elastic Beanstalk
## I. Setup Django and S3 File Storage
First, we need to setup our S3 File Storage to serve our static files and media files in AWS.
### Create AWS Account
### Create AWS User Credentials
1. Navigate to IAM Users
2. Select ***Create New Users***
3. Enter a user name
4. Ensure ***Prammatic Access*** is selected, hit ***Next***.
5. Select ***Create a Group***
6. Define a name for the group and search for built-in policy ***AmazonS3FullAccess***, hit ***Create group***
7. Select ***Next: Review***
8. Select ***Download credentials*** and keep them safe.
9. Open the credentials.csv file that was just downloaded/created
10. Note the Access Key Id and Secret Access Key as they are needed for a future step. These will be referred to as ***<your_access_key_id>*** and ***<your_secret_access_key>***
    
### Create S3 Bucket
1. Create a bucket and set a name for your bucket (usually app name)
2. Select the region closest to your area: ***Southeast***
3. Click next until you finish and leave all as is
    
### Add Default Access Policies to your IAM User
1. Navigate to IAM Home
2. Select user and click on the ***Permissions*** tab
3. Click ***Add Permissions***
4. Click on ***Attach Existing Policies Directly***
5. Click ***Create Policy*** then select ***Create Your Own Policy***
6. Set Policy Name
7.  Set Policy Document. Make sure to change the S3 bucket names
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets"
            ],
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation",
                "s3:ListBucketMultipartUploads",
                "s3:ListBucketVersions"
            ],
            "Resource": "arn:aws:s3:::<your_bucket_name>"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:*Object*",
                "s3:ListMultipartUploadParts",
                "s3:AbortMultipartUpload"
            ],
            "Resource": "arn:aws:s3:::<your_bucket_name>/*"
        }
    ]
}
```
### Add a CORS Policy to bucket
1. Click on bucket
2. Permissions > Cors Configuration
3. Paste:
```
<CORSConfiguration>
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
        <MaxAgeSeconds>3000</MaxAgeSeconds>
        <AllowedHeader>Authorization</AllowedHeader>
    </CORSRule>
</CORSConfiguration>
```
### Setup Django
1. Install Django Storages: 
``` python -m pip install boto boto3 django-storages ```
2. Update INSTALLED APPS in settings.py:
```    
INSTALLED_APPS = [
    ...
    'storages',
    ...
]
```
3. Migrate: 
``` python manage.py migrate ```
4. Create and go to ***aws*** folder in same directory as settings.py
5. Create three files in the 'aws' folder: ***'__init__.py', 'utils.py', 'conf.py '***
6. In ***utils.py*** add:
```
from storages.backends.s3boto3 import S3Boto3Storage

StaticRootS3BotoStorage = lambda: S3Boto3Storage(location='static')
MediaRootS3BotoStorage  = lambda: S3Boto3Storage(location='media')
```
7. In ***conf.py*** add:
```
import datetime
AWS_ACCESS_KEY_ID = "<your_access_key_id>"
AWS_SECRET_ACCESS_KEY = "<your_secret_access_key>"
AWS_FILE_EXPIRE = 200
AWS_PRELOAD_METADATA = True
AWS_QUERYSTRING_AUTH = True
AWS_DEFAULT_ACL = 'private'

DEFAULT_FILE_STORAGE = '<your-project>.aws.utils.MediaRootS3BotoStorage'
STATICFILES_STORAGE = '<your-project>.aws.utils.StaticRootS3BotoStorage'
AWS_STORAGE_BUCKET_NAME = '<your_bucket_name>'
S3DIRECT_REGION = 'ap-southeast-1'
S3_URL = '//%s.s3.amazonaws.com/' % AWS_STORAGE_BUCKET_NAME
MEDIA_URL = '//%s.s3.amazonaws.com/media/' % AWS_STORAGE_BUCKET_NAME
MEDIA_ROOT = MEDIA_URL
STATIC_URL = S3_URL + 'static/'
ADMIN_MEDIA_PREFIX = STATIC_URL + 'admin/'

two_months = datetime.timedelta(days=61)
date_two_months_later = datetime.date.today() + two_months
expires = date_two_months_later.strftime("%A, %d %B %Y 20:00:00 GMT")

AWS_HEADERS = { 
    'Expires': expires,
    'Cache-Control': 'max-age=%d' % (int(two_months.total_seconds()), ),
}
```
8. In ***setttings.py*** add:
```
from <your-project>.aws.conf import *
```
9. Run:
```
python manage.py collectstatic
```
After this, you can check your application
## II. Setup Django and Elastic Beanstalk
Next, we host our application to AWS Elastic Beanstalk. Elastic Beanstalk is a service to deploy, manage and scale applications in different languages. With this, we do not have to worry about scaling our infrastructure based on the demand and traffic of our application. Hence, it reduces infrastructure management complexities without affecting our choice and control of which service and technologies to use.
### Configure Django with PostgreSQL
1. Create a file for all modules used
``` python -m pip freeze > requirements.txt ```
2. Create a directory called ***.ebextensions*** under the same directory/root of your project folder
3. Add a configuration file called ***django.config*** and paste the following:
```
container_commands:
  01_migrate:
    command: "python manage.py migrate"
    leader_only: true
  02_collectstatic:   
    command: "python manage.py collectstatic --noinput"
option_settings:
  aws:elasticbeanstalk:container:python:
    WSGIPath: <project-name>/wsgi.py
    
```
if you have an SSL certificate included add the following as well:
```
files:
  "/etc/httpd/conf.d/ssl_rewrite.conf":
      mode: "000644"
      owner: root
      group: root
      content: |
          RewriteEngine On
          <If "-n '%{HTTP:X-Forwarded-Proto}' && %{HTTP:X-Forwarded-Proto} != 'https'">
          RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R,L]
          </If>
```
4. Add another configuration file called ***packages.config*** and paste the following:
```
packages:
  yum:
    postgresql93-devel: []
```
You can view the full documentation of Justin Mitchel [here](https://www.codingforentrepreneurs.com/blog/s3-static-media-files-for-django).
### Create your Elastic Beanstalk Environment
1. Install your eb command line (eb cli)
```
python -m pip install awsebcli
eb --version
```
2. Initialize eb cli.
```
eb init 
```
3. Provide your access keys to be able to manage your AWS resources
```
You have not yet set up your credentials or your credentials are incorrect.
You must provide your credentials.
(aws-access-id): AKIAJOUAASEXAMPLE
(aws-secret-key): 5ZRIrtTM4ciIAvd4EXAMPLEDtm+PiPSzpoK
```
4. Set your application name, platform and SSH key pair. You can use the default **'eb'** name for you application. Next, choose which platform you're using. In our case, choose **Python 3.6**. Lastly, use the default option for your key pair. If you want to change your settings, you can use ``` eb init -i ```. More documentation for configurations are included in the [AWS Documentation](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create-deploy-python-django.html).

5. Create your environment with your project name``` eb create ``` This command will create your environment with the name you've specified.
6. Check your environment status ``` eb status ``` and copy the CNAME of your environment.
7. Copy the CNAME and add this to ALLOWED_HOSTS in your settings.py
```
ALLOWED_HOSTS = [..., 'project-name.elasticbeanstalk.com']
```
8. Deploy your latest commits using ```eb deploy```. Make sure you've already commited your latest changes to your master branch.
9. Check if your app is already running in your environment ```eb open```. You can also track your environment status in your Elastic Beanstalk Console in AWS.
### Setup your Database Server for PostgreSQL
1. Change your database settings in ***settings.py***
```
# Database
# https://docs.djangoproject.com/en/2.2/ref/settings/#databases

if 'RDS_DB_NAME' in os.environ:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': os.environ['RDS_DB_NAME'],
            'USER': os.environ['RDS_USERNAME'],
            'PASSWORD': os.environ['RDS_PASSWORD'],
            'HOST': os.environ['RDS_HOSTNAME'],
            'PORT': os.environ['RDS_PORT'],
        }
    }
else:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
        }
    }
```
2. In ***settings.py***, change ```DEBUG = True``` to ```DEBUG = False```.
3. Run the command ```eb console```
4. In the menu, select ***Configurations > Databases > Modify***
5. Pick a number from 5 to 1024 to select the storage capacity of your database.
6. Create a username and password then Click ***Apply***
