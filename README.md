### Setting up the Apache Superset custom build

1. Clone this repository https://github.com/DGM-Tech-Solutions/superset.git
2. replace the .env file inside the docker directory

```env
COMPOSE_PROJECT_NAME=superset

# database configurations (do not modify)
DATABASE_DB=superset
DATABASE_HOST=db
# Make sure you set this to a unique secure random value on production
DATABASE_PASSWORD=superset
DATABASE_USER=superset

EXAMPLES_DB=examples
EXAMPLES_HOST=db
EXAMPLES_USER=examples
# Make sure you set this to a unique secure random value on production
EXAMPLES_PASSWORD=examples
EXAMPLES_PORT=5432

# database engine specific environment variables
# change the below if you prefer another database engine
DATABASE_PORT=5432
DATABASE_DIALECT=postgresql
POSTGRES_DB=superset
POSTGRES_USER=superset
# Make sure you set this to a unique secure random value on production
POSTGRES_PASSWORD=superset
#MYSQL_DATABASE=superset
#MYSQL_USER=superset
#MYSQL_PASSWORD=superset
#MYSQL_RANDOM_ROOT_PASSWORD=yes

# Add the mapped in /app/pythonpath_docker which allows devs to override stuff
PYTHONPATH=/app/pythonpath:/app/docker/pythonpath_dev
REDIS_HOST=redis
REDIS_PORT=6379

FLASK_DEBUG=true
SUPERSET_ENV=production
SUPERSET_LOAD_EXAMPLES=yes
CYPRESS_CONFIG=false
SUPERSET_PORT=8088
MAPBOX_API_KEY=''

# Make sure you set this to a unique secure random value on production
SUPERSET_SECRET_KEY=TEST_NON_DEV_SECRET

ENABLE_PLAYWRIGHT=false
PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
BUILD_SUPERSET_FRONTEND_IN_DOCKER=true
```
3.  Now replace the` superset_config.py` file in this directory  or else delete the file using 
   `rm -rf` command and then creating the new one IN this directory `docker/pythonpath_dev/superset_config.py`

```python
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#
# This file is included in the final Docker image and SHOULD be overridden when
# deploying the image to prod. Settings configured here are intended for use in local
# development environments. Also note that superset_config_docker.py is imported
# as a final step as a means to override "defaults" configured here
#
import logging
import os
import re
import sys
from collections import OrderedDict
from datetime import timedelta
from email.mime.multipart import MIMEMultipart
from typing import Any, Callable, Literal, TYPE_CHECKING, TypedDict


from celery.schedules import crontab
from flask_caching.backends.filesystemcache import FileSystemCache

logger = logging.getLogger()

FAB_ADD_SECURITY_API = True

DATABASE_DIALECT = os.getenv("DATABASE_DIALECT")
DATABASE_USER = os.getenv("DATABASE_USER")
DATABASE_PASSWORD = os.getenv("DATABASE_PASSWORD")
DATABASE_HOST = os.getenv("DATABASE_HOST")
DATABASE_PORT = os.getenv("DATABASE_PORT")
DATABASE_DB = os.getenv("DATABASE_DB")

EXAMPLES_USER = os.getenv("EXAMPLES_USER")
EXAMPLES_PASSWORD = os.getenv("EXAMPLES_PASSWORD")
EXAMPLES_HOST = os.getenv("EXAMPLES_HOST")
EXAMPLES_PORT = os.getenv("EXAMPLES_PORT")
EXAMPLES_DB = os.getenv("EXAMPLES_DB")

# The SQLAlchemy connection string.
SQLALCHEMY_DATABASE_URI = (
    f"{DATABASE_DIALECT}://"
    f"{DATABASE_USER}:{DATABASE_PASSWORD}@"
    f"{DATABASE_HOST}:{DATABASE_PORT}/{DATABASE_DB}"
)

SQLALCHEMY_EXAMPLES_URI = (
    f"{DATABASE_DIALECT}://"
    f"{EXAMPLES_USER}:{EXAMPLES_PASSWORD}@"
    f"{EXAMPLES_HOST}:{EXAMPLES_PORT}/{EXAMPLES_DB}"
)

REDIS_HOST = os.getenv("REDIS_HOST", "redis")
REDIS_PORT = os.getenv("REDIS_PORT", "6379")
REDIS_CELERY_DB = os.getenv("REDIS_CELERY_DB", "0")
REDIS_RESULTS_DB = os.getenv("REDIS_RESULTS_DB", "1")

RESULTS_BACKEND = FileSystemCache("/app/superset_home/sqllab")

CACHE_CONFIG = {
    "CACHE_TYPE": "RedisCache",
    "CACHE_DEFAULT_TIMEOUT": 300,
    "CACHE_KEY_PREFIX": "superset_",
    "CACHE_REDIS_HOST": REDIS_HOST,
    "CACHE_REDIS_PORT": REDIS_PORT,
    "CACHE_REDIS_DB": REDIS_RESULTS_DB,
}
DATA_CACHE_CONFIG = CACHE_CONFIG


GLOBAL_ASYNC_QUERIES_REDIS_CONFIG = {
    "port": 6379,
    "host": REDIS_HOST,
    "password": "",
    "db": 0,
    "ssl": False,
}
GLOBAL_ASYNC_QUERIES_REDIS_STREAM_PREFIX = "async-events-"
GLOBAL_ASYNC_QUERIES_REDIS_STREAM_LIMIT = 1000
GLOBAL_ASYNC_QUERIES_REDIS_STREAM_LIMIT_FIREHOSE = 1000000
GLOBAL_ASYNC_QUERIES_JWT_COOKIE_NAME = "async-token"
GLOBAL_ASYNC_QUERIES_JWT_COOKIE_SECURE = False
GLOBAL_ASYNC_QUERIES_JWT_COOKIE_SAMESITE: None
GLOBAL_ASYNC_QUERIES_JWT_COOKIE_DOMAIN = None
GLOBAL_ASYNC_QUERIES_JWT_SECRET = "="
GLOBAL_ASYNC_QUERIES_TRANSPORT = "polling"
GLOBAL_ASYNC_QUERIES_POLLING_DELAY = int(
    timedelta(milliseconds=500).total_seconds() * 1000
)


class CeleryConfig:
    broker_url = f"redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_CELERY_DB}"
    imports = ("superset.sql_lab","superset.tasks",)
    result_backend = f"redis://{REDIS_HOST}:{REDIS_PORT}/{REDIS_RESULTS_DB}"
    worker_prefetch_multiplier = 3
    task_acks_late = True
    beat_schedule = {
        "reports.scheduler": {
            "task": "reports.scheduler",
            "schedule": crontab(minute="*", hour="*"),
        },
        "reports.prune_log": {
            "task": "reports.prune_log",
            "schedule": crontab(minute=10, hour=0),
        },
    }


CELERY_CONFIG = CeleryConfig

FEATURE_FLAGS = {
    "ALERT_REPORTS": True,
    "ENABLE_TEMPLATE_PROCESSING" : True,
    "DASHBOARD_CROSS_FILTERS" : True,
    "HORIZONTAL_FILTER_BAR": True,
    "DASHBOARD_FILTERS_EXPERIMENTAL": True,
    "DASHBOARD_NATIVE_FILTERS_SET": True,
    "DASHBOARD_NATIVE_FILTERS": True,
    "LISTVIEWS_DEFAULT_CARD_VIEW": True,
    "GLOBAL_ASYNC_QUERIES":True,
    "DASHBOARD_RBAC": True,
    }
ALERT_REPORTS_NOTIFICATION_DRY_RUN = False
WEBDRIVER_BASEURL = "http://superset:8088/"
# The base URL for the email report hyperlinks.
WEBDRIVER_BASEURL_USER_FRIENDLY = WEBDRIVER_BASEURL

SQLLAB_CTAS_NO_LIMIT = True

#

# Optionally import superset_config_docker.py (which will have been included on
# the PYTHONPATH) in order to allow for local settings to be overridden
#
try:
    import superset_config_docker
    from superset_config_docker import *  # noqa

    logger.info(
        f"Loaded your Docker configuration at " f"[{superset_config_docker.__file__}]"
    )
except ImportError:
    logger.info("Using default Docker config...")

TALISMAN_ENABLED = False

APP_NAME = "Neotrades BI Dashboard"



# Uncomment to setup an App icon
APP_ICON = "https://ergodotisi.blob.core.windows.net/generalimages/companies/33227.jpg"
APP_ICON_WIDTH = 150

FAVICONS = [{"href": "https://lh3.googleusercontent.com/-F-FzCn3BpNU/AAAAAAAAAAI/AAAAAAAAAAA/pRUX-cvC5P8/s88-w44-h44-p-k-no-ns-nd/photo.jpg"}]

SCREENSHOT_LOCATE_WAIT = 100
SCREENSHOT_LOAD_WAIT = 600

# Slack configuration
SLACK_API_TOKEN = "xoxb-token"

# Email configuration
SMTP_HOST = "smtp.strato.de" # change to your host
SMTP_PORT = 587 # your port, e.g. 587
SMTP_STARTTLS = True
SMTP_SSL_SERVER_AUTH = False # If your using an SMTP server with a valid certificate
SMTP_SSL = False
SMTP_USER = "no-reply@kexample.com" # use the empty string "" if using an unauthenticated SMTP server
SMTP_PASSWORD = "" # use the empty string "" if using an unauthenticated SMTP server   --------------------------CHANGE THIS----------------------
SMTP_MAIL_FROM = "no-reply@kexample.com"
EMAIL_REPORTS_SUBJECT_PREFIX = "[RMS CRM Alert] " # optional - overwrites default value in config.py of "[Report]
EMAIL_REPORTS_CTA = "Explore in Dashboard"


SUPERSET_WEBSERVER_TIMEOUT = int(timedelta(minutes=5).total_seconds())

```

4. Now go back the the `superset` directory
5.  Now run the `docker-compose-non-dev.yml` file using the docker compose command `docker compose -f docker-compose-non-dev.yml up -d`
