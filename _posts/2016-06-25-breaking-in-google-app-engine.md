---
layout: post
comments: true
title: Breaking in Google App Engine
preview: This post is specifically on Google App Engine, and goes over deploying a simple web application from scratch.
---

Google Cloud Platform is Google's solution to cloud hosting.  Initially it was released as Google App Engine back in 2008, but Google found that developers weren't flocking to their platform as anticipated due to how different Google was doing things.  In 2011 Google Cloud Platform was launched which included Google App Engine and Google Cloud Storage.  As the years progressed Google had to create solutions that matched how developers were working today, this is where other products such as Google Cloud Engine came along.

This post is specifically on Google App Engine, and goes over deploying a simple web application from scratch.

## Setting up

First you will need to create a new project by going to [console.cloud.google.com](https://console.cloud.google.com).  After logging in click the accordion at the top left and select IAM & Admin.  From hear select "Create Project" and give your project a name.

After the project has been created click the accordion again in the top left and select "Development".  Make sure that "Source Code" is selected along the left, and then select "Get started" on the pop-up in the middle.  In this walkthrough I opt for "Automatically mirror from GitHub or Bitbucket" and provide Google with access to my private bitbucket repository.

At this point I recommend following the linked instructions to set up Google Cloud's SDK at [cloud.google.com/sdk/](https://cloud.google.com/sdk/).  You will also need to install the Python libraries with `gcloud components install app-engine-python`, along with the python SDK at [cloud.google.com/appengine/downloads](https://cloud.google.com/appengine/downloads).

## Creating the app

You'll want to first clone your code onto your local machine.  For me the command was similar to `git clone https://username@bitbucket.org/user/gcp-demo-app.git gcp-demo-app`.

Inside of the cloned repository you will need to make an `app.yaml` file that defines the runtime environment for your application.  An example configuration for a basic flask app looks like the following:

```
runtime: python27
api_version: 1
threadsafe: true

# [START handlers]
handlers:
- url: /.*
  script: gcp.app
# [END handlers]

```

This should all be pretty straight forward, for a full reference if you don't understand anything check out their [documentation](https://cloud.google.com/appengine/docs/python/config/appref#syntax).

Next we'll need to create a `gcp.py` file to house our `app` callable like we referenced in the `app.yaml` file.  Using Flask a simple example looks as follows:

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"
```

Google is smart enough to do `app.run()` for us, and so that is omitted.

## Vendoring Flask

Since Flask isn't in the standard library for Python this adds some small complications, Google App Engine requires that any imported libraries exist in our applications folder.

Let's update `gcp.py` to look like the following at the top:

```python
from google.appengine.ext import vendor

# Add any libraries installed in the "lib" folder.
vendor.add('lib')
<...>
```

Let's also create a `requirements.txt` file and add the following to it:

```
flask>=0.11
```

Now Google App Engine knows to look inside of the `lib` folder relative to our applications location.  Let's go ahead and install Flask inside of the lib folder like so:

```bash
mkdir lib
pip install -t lib -r requirements.txt
```

## Testing our work

Before you push the code up to your selected SCM website you can run the application locally to see that everything works.  You do this with the Google Cloud SDK by running `dev_appserver.py .`.  This command runs the application like it will in Google App Engine from your current directory.

If the command starts successfully you should be able to visit your browser at `http://localhost:8080` and see your application running.

## Deployment

Since we're using SCM this is pretty straight-forward.  First push your code up:

```bash
git add app.yaml gcp.py lib requirements.txt
git commit -m "Initial commit"
git push
```

On the  Google Cloud Console you should see your code populate into their website under the "Source Code" section.

Finally run this locally:

```bash
appcfg.py -A [YOUR_PROJECT_ID] -V v1 update ./
```

After the command completes you should see your application deployed at `https://<project-id>.appspot.com`.
