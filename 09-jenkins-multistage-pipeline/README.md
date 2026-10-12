# Exercise 9: Jenkins Multi-Stage Pipeline (Python Flask app)

## Objective
Build a Jenkins pipeline that runs several stages (build, test, deploy, run, test the app) for a Flask application, with the Jenkinsfile pulled from Git. I also did all three extras: a flake8 code quality stage, a Docker deploy stage, and email notifications.

## Environment
Windows 11, Docker Desktop, Jenkins 2.580.1 in the container jenkins7 (Debian 13, Python 3.13.5, pip 25.1.1, Flask 3.1.1 from apt, flake8 7.1.1). The code is in my repository https://github.com/ankursaiis23-ctrl/devops-sample-code (the same one as Exercise 8).

## Files in this folder
| File | Purpose |
|---|---|
| app.py, test_app.py, requirements.txt | The Flask app and its unit test |
| Jenkinsfile | Final pipeline (all stages and mail notifications) |
| Dockerfile | Image for the Docker deploy extra |
| console-output.txt | Log of build 1 (base pipeline) |
| console-output-build2 to build6 | Logs of the later builds |

## Changes I made to the exercise, and why
- **Existing repo:** devops-sample-code already had hello-world.sh from Exercise 8, so I added the files to my existing clone instead of running git init again.
- **Branch:** Pipeline script from SCM defaults to */master. I changed it to */main.
- **Python in Jenkins:** the exercise installs python3, pip and python3.11-venv. Python 3.13.5 was already in the image, and python3.11-venv does not exist on Debian 13, so I installed python3-venv, python3-pip and python3-flask.
- **Flask versions:** requirements.txt pins flask==2.1.2, but the Build stage only prints a message and does not install it. The tests ran against Flask 3.1.1 from apt.
- **Deploy location:** the exercise says app.py is copied to /tmp/python-app-deploy. The Jenkinsfile actually copies it to python-app-deploy inside the workspace, and the log shows that.
- **Stages shown:** Jenkins shows 7 stages for the base pipeline, not 5, because declarative pipelines add Checkout SCM and Post Actions.
- **Test Application:** this stage reruns test_app.py. It does not send a request to the running app, so it does not prove the app is up.
## Base pipeline
I started the Jenkins container jenkins7 and created the exercise folder.

![jenkins started](jenkins-started.png)

I created the Flask app, requirements.txt and the unit test in my devops-sample-code clone and pushed them.

![flask files created](flask-files-created.png)
![flask push](flask-push.png)

Then I added the Jenkinsfile and pushed it.

![jenkinsfile push](jenkinsfile-push.png)

Python was checked and installed in Jenkins first.

![python check](jenkins-python-check.png)
![python installed](jenkins-python-installed.png)

The job Python-MultiStage-Pipeline uses Pipeline script from SCM (Git, branch */main, script path Jenkinsfile).

![config 1](pipeline-job-config-1.png)
![config 2](pipeline-job-config-2.png)

Build 1 passed: Test ran 1 test OK, app.py was copied, started in the background and tested.

![console](pipeline-console-output.png)
![stages](pipeline-stage-view.png)

## Extra 1: Code quality (flake8)
flake8 was installed in Jenkins with apt. Running it on the exercise code found 5 problems (counted once, the deploy copy repeats the app.py ones): E302 and E305 (needs 2 blank lines) in app.py and test_app.py, and E501 (line too long, 95 > 79) in test_app.py. I fixed the files and checked them with flake8 before pushing.

![installed](flake8-installed.png)
![errors](flake8-errors.png)
![clean](flake8-clean.png)

I pushed the fixed files together with the updated Jenkinsfile.

![code quality push](codequality-push.png)

I added a Code Quality stage after Test that runs `flake8 --exclude=python-app-deploy .`. The exclusion is needed because the workspace still holds an old copy of app.py in python-app-deploy from build 1, and flake8 would fail on that stale copy. Build 2 passed with 8 stages.

![stages](codequality-stage-view.png)

## Extra 2: Deploy in a Docker container
I added a Dockerfile and a Docker Deploy stage that builds the image, removes any old container, and runs it with `-p 5001:5000`. Changes needed: app.run uses host="0.0.0.0" (otherwise the app only listens inside the container), the image installs plain flask instead of the old pin, and port 5001 avoids clashes. Build 3 passed, and the app answers from my PC.

![docker check](docker-deploy-check.png)
## Extra 3: Email notifications
I used Mailpit, a fake mail server in a container, so no real account or password is needed. Jenkins sends to host.docker.internal on port 1025 (the same address trick as Exercise 6), and the mails show at http://localhost:8025. The test mail from Manage Jenkins arrived.

![mail config](jenkins-mail-config.png)
![mailpit](mailpit-running.png)
![test mail](mailpit-test-mail.png)

The post block now sends a mail with the mail step on success and on failure. To prove the failure mail, I added the line x=1 to app.py on purpose. Build 5 failed in Code Quality with E225 (missing whitespace around operator), the later stages were skipped, and the FAILED mail arrived. I reverted the commit and build 6 passed again.

![mails](mailpit-success-failure.png)
![stages](notify-stage-view.png)

Builds: 1 base, 2 flake8, 3 Docker deploy, 4 mail on success, 5 deliberate failure, 6 recovery. Logs are in the console-output files. The Slack option was not used, because it needs a workspace and a secret webhook URL.

## What I learned
- A multi-stage pipeline stops at the first failing stage and skips the rest, but the post block still runs, so a notification can be sent.
- Running flake8 in the pipeline turns style rules into a gate: the exercise code failed it until I fixed the blank lines and the long line.
- Jenkins inside a container uses the host's Docker through the socket, and the app in a new container needs host 0.0.0.0 to be reachable.
- The exercise's package names and paths did not match my Jenkins image, so I checked the real versions before installing.