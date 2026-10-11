# Exercise 6: Real-Time Operations Monitoring and Alerting

## Objective
Simulate delivery metrics with Python, scrape them with Prometheus, show them in Grafana, raise alerts, and automate the setup with a Jenkins pipeline.

## Files
| File | Purpose |
|---|---|
| delivery_metrics.py | Simulates delivery metrics on port 8000 (currently the Step 6 version, pending = randint(50, 100)) |
| prometheus.yml | Scrape config (targets host.docker.internal:8000) |
| alert_rules.yml | HighPendingDeliveries and HighAverageDeliveryTime alerts |
| Dockerfile | Added by me, because the Jenkinsfile runs docker build and the exercise gives none |
| Jenkinsfile | Six stage pipeline, adjusted for Docker Desktop on Windows |
| jenkins-console-output.txt | Full console log of build #1 (ends in Finished: SUCCESS) |

## Environment
Windows 11, Docker Desktop 29.7.2, Python 3.14.3, prometheus-client 0.26.0, Jenkins 2.580.1 in Docker.

## Changes I made to the exercise, and why
1. **host.docker.internal instead of 172.17.0.1.** 172.17.0.1 is the Linux docker0 address. Docker Desktop on Windows has no docker0 on the host, and `ip addr show docker0` does not exist. I used `host.docker.internal` for the scrape target and the Grafana data source. A ping from a container resolved it to 192.168.65.254.
2. **File name.** The exercise lists alerts_rules.yml but prometheus.yml and the docker run command use alert_rules.yml, so I used alert_rules.yml everywhere.
3. **Port publishing instead of --network=host.** Host networking does not work like Linux on Docker Desktop, so Prometheus runs with `-p 9090:9090`.
4. **Windows commands.** `python` and `python -m pip` instead of python3 and pip (pip was not on PATH), and `curl.exe` instead of curl.
5. **Jenkins needs Docker inside it.** The stock image has no Docker CLI, so the Pre-check stage would fail. I started Jenkins as root with the Docker socket mounted, mounted this folder at /src, and installed the Docker CLI with apt-get (docker.io).
6. **Initial password.** The exercise reads it with `sudo cat /var/lib/jenkins/...`, which is the wrong path with no sudo. I used `docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`. The password is not in this repo.
7. **Jenkinsfile.** /path/to/your/local/files became /src. Mounting $WORKSPACE into Prometheus does not work, because that path exists inside the Jenkins container and not for the Docker daemon, so the pipeline uses docker create, docker cp and docker start. I added a Clean Old Containers stage so the pipeline can be run again without name clashes.

## Part 1: Metrics, Prometheus, alerts
Tools and library:

![tools](tools-check.png)
![install](install-prometheus-client.png)

The script runs and serves /metrics:

![script running](script-running.png)
![metrics endpoint](metrics-endpoint.png)

Config files and Prometheus in Docker:

![config](config-files.png)
![prometheus running](prometheus-running.png)
![host address](host-address-check.png)
![targets](prometheus-targets.png)

Queries and alerts. HighPendingDeliveries fires with values between 10 and 20. HighAverageDeliveryTime stayed inactive because the running average sat at about 29.9, just under 30.

![pending query](prometheus-query-pending.png)
![average query](prometheus-query-avgtime.png)
![alerts](prometheus-alerts.png)

## Part 2: Grafana
Data source `http://host.docker.internal:9090` and a four panel dashboard (Total, Pending, On-the-Way, Average Delivery Time).

![grafana running](grafana-running.png)
![datasource url](grafana-datasource-url.png)
![datasource ok](grafana-datasource.png)
![panel query](grafana-panel-query.png)
![dashboard](grafana-dashboard.png)

Grafana suggested using rate() on the average query because the metric is a counter. The exercise formula is sum / count, which gives the average since the script started, so I kept it.

## Part 3: Step 6, simulating alerts
I changed pending to randint(50, 100), restarted the script and Prometheus. HighPendingDeliveries fired with a value of 88. The dashboard shows the jump, and a gap while the script was stopped.

![script modified](script-modified.png)
![alert fired](prometheus-alerts-fired.png)
![dashboard after](grafana-dashboard-after.png)

## Part 4: Jenkins pipeline
Jenkins runs in Docker with access to the host Docker through the socket. Jenkins shows times in UTC, so build #1 shows 3:06 AM, which is 8:36 local.

![dockerfile and jenkinsfile](jenkins-files-created.png)
![jenkins running](jenkins-running.png)
![docker access](jenkins-docker-access.png)
![jenkins home](jenkins-home.png)
![pipeline created](jenkins-pipeline-created.png)
![config 1](jenkins-pipeline-config-1.png)
![config 2](jenkins-pipeline-config-2.png)
![config 3](jenkins-pipeline-config-3.png)
![config 4](jenkins-pipeline-config-4.png)
![config 5](jenkins-pipeline-config-5.png)

Build #1 succeeded in 22 seconds. All six stages passed. The line `No such container: delivery_metrics` in the Clean stage is expected, because that container did not exist yet and the stage uses `|| true`. The full log is in jenkins-console-output.txt.

![success](jenkins-pipeline-success.png)
![stage view](jenkins-stage-view.png)
![log check](jenkins-console-check.png)
![containers](pipeline-containers.png)
![pipeline targets](pipeline-prometheus-targets.png)

The Grafana started by the pipeline is a fresh container, so the data source and dashboard from Part 2 are not kept. I did not mount a volume, because the exercise only asks the pipeline to start the services.

## What I learned
- A monitoring setup has three parts: the app exposes metrics, Prometheus scrapes them and evaluates alert rules, and Grafana draws them.
- Commands written for Linux often need changing on Windows. The host address was the main one, and I checked it with a ping instead of assuming.
- Jenkins in a container cannot run Docker commands unless it has the Docker CLI and access to the host's Docker socket. Paths inside the Jenkins container do not exist for the Docker daemon, which is why I used docker cp.
- An alert that sits near its threshold can flicker. Raising the pending range made the alert fire clearly.