# Part Three - building a real pipeline

## Fork repo
Go to GitHub: [https://github.com/Sharor/concourse-workshop](https://github.com/Sharor/concourse-workshop)

Fork the repository into your own GitHub account.

Clone that fork locally with:

```
git clone https://github.com/<YOUR_USER>/concourse-workshop
```

## Create token or use username/password credentials

If you want to use a single use access token, you can create one easily with:
```sh
curl -i -u <GITHUB_USER> \
-d '{"scopes": ["repo", "user", "gist"], "note": "concourse-meetup"}' \
https://api.github.com/authorizations
```

or through the web-interface.

Alternatively you can do the exercise using your normal credentials.

## Configure credentials 
Open the `credentials.yml` file in your favourite editor.

Insert your username in the username field.

Insert either your password or your generated token in the password field.


## Log in to the concourse
You should already be logged in from previous exercises, otherwise use:

```sh
fly -t meetup login -c http://35.156.129.15/
```



## Set up the pipeline:
```sh
fly set-pipeline -t meetup -c pipeline.yml -p [YOUR_PIPELINE_NAME] -l credentials.yml
```

e.g.
```sh
fly set-pipeline -t meetup -c pipeline.yml -p peter-jekyll -l credentials.yml
```

### This will NOT work 
Investigate  why

## Edit pipeline.yml to make it work!
* Change links to point to your own fork
* Create a new gist in your own github account
    - create a single markdown file with some name, e.g. foo.md
    - Copy the https url for the gist into the pipeline.yml






