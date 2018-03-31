# Firebase tools docker

Setup a Firebase cli tools environment inside a docker container

## 1. Build the `Dockerfile` and name the image

The docker image will be created from the basic node:alpine image

```
FROM node:9.10-alpine

EXPOSE 9005

RUN npm install -g firebase-tools
```

It's important to name the resulting image, as the name will be used later. 
In this case I'm naming it `"lgvalle/firebase-tools-docker"`

```bash
> docker build -t lgvalle/firebase-tools-docker .
```

This will create an image like this

```
luis@mbp > docker images

REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
lgvalle/firebase-tools-docker   latest              dc8673fe6ec9        18 minutes ago      152MB
```

## 2. Bash into the container and login with Firebase

`> docker run -p 9005:9005 -u node -it lgvalle/firebase-tools-docker sh`

This command executes `sh` (effectively opening a `sh` console) in the image `lgvalle/firebase-tools-docker` but also:

  * Maps the host's port `9005` to the corresponding container port. This is important for Firebase login as the oauth callback uses this port
  * Sets the username as `node` - we want to avoid running stuff as `root` if possible

If all goes well a terminal within the container should be opened

```
/ $ firebase login
? Allow Firebase to collect anonymous CLI usage and error reporting information? Yes

Visit this URL on any device to log in:
https://accounts.google.com/o/oauth2/auth?client_id=xxxxxo849e6.apps.googleusercontent.com&scope=email%20openid%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloudplatformprojects.readonly%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Ffirebase%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform&response_type=code&state=176507547&redirect_uri=http%3A%2F%2Flocalhost%3A9005

Waiting for authentication...
``` 

Copy & paste that url into a browser. A Google auth window should appear. 
Complete the login process and go back to the terminal. A message like this will be printed if everything goes well:

```
Waiting for authentication...

✔  Success! Logged in as luis@novoda.com
```

## 3. Commit container changes as new image

Everytime you spin an image, docker creates a new container based in that image to execute the work. Any state changes inside the container are lost if you don't commit them

> This is not technically correct, as you can always re-run an old container. In this case, what we want is to have a docker image with our user already logged in; and for that, we need to commit the changes occurred during the login process 

Exit the container typing `exit`, find the container id and commit the changes:

```
luis@mbp > docker ps -l                                                                                                  
CONTAINER ID        IMAGE                                 COMMAND             CREATED             STATUS                      PORTS               NAMES
f1ac27f6fa4e        lgvalle/firebase-tools-docker   "sh"                42 minutes ago      Exited (0) 33 minutes ago                       youthful_saha


luis@mbp > docker commit --message "Add firebase login" f1ac27f6fa4e lgvalle/firebase-tools-docker
```

This will replace our original `lgvalle/firebase-tools-docker` image with the content of the container `f1ac27f6fa4e` 

## 4. Verify the new image has valid login details


Bash into the container and list your firebase projects

```
luis@mbp > docker run -p 9005:9005 -u node -it lgvalle/firebase-tools-docker sh

/ $ firebase list

/usr/app $ firebase list
┌───────────────────────────┬─────────────────────────────┬─────────────┐
│ Name                      │ Project ID / Instance       │ Permissions │
├───────────────────────────┼─────────────────────────────┼─────────────┤
│ bonfire                   │ bonfire-b4111               │ Editor      │
├───────────────────────────┼─────────────────────────────┼─────────────┤
│ Firebase Demo Project     │ fir-demo-project-ad555      │ Viewer      │
├───────────────────────────┼─────────────────────────────┼─────────────┤
│ FirebaseNotificationsDemo │ fir-notificationsdemo-66111 │ Owner       │
├───────────────────────────┼─────────────────────────────┼─────────────┤
│ MyAwesomeProject          │ myawesomeproject-771111     │ Owner       │
└───────────────────────────┴─────────────────────────────┴─────────────┘
```

If a list of projects like this is shown it means everything worked well

## 5. Bonus: launch firebase tools environment for the current directory

Create an alias like this to automatically open a firebase tools console for the current directory:

```bash
alias fire='docker run -v $PWD:/usr/app -w /usr/app -p 9005:9005 -u node -it lgvalle/firebase-tools-docker sh' 
```

* `-v $PWD:/usr/app` : maps current directory to the path `/usr/app` inside the container
* `-w /usr/app` : sets the container working dir to `usr/app`


