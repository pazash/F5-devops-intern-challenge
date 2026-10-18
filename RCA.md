# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
Error on installing the packages:
target proxy: failed to solve: process "/bin/sh -c apt-get update &&     apt-get install -y nginx curl htop vim git" did not complete successfully: exit code: 100

Above it, dependancy error while trying to install nginx:
29.54 dpkg: dependency problems prevent configuration of nginx:
29.54  nginx depends on nginx-common (= 1.24.0-2ubuntu7.18); however:
29.54   Package nginx-common is not configured yet.

Above it, error while trying to install nginx-common:
28.19 Configuration file '/etc/nginx/nginx.conf'
28.19  ==> File on system created by you or by a script.
28.19  ==> File also in package provided by package maintainer.
28.19    What would you like to do about it ?  Your options are:
28.19     Y or I  : install the package maintainer's version
28.19     N or O  : keep your currently-installed version
28.19       D     : show the differences between the versions
28.19       Z     : start a shell to examine the situation
28.19  The default action is to keep your current version.
28.19 *** nginx.conf (Y/I/N/O/D/Z) [default=N] ? dpkg: error processing package nginx-common (--configure):
28.19  end of file on stdin at conffile prompt


### How I diagnosed it
I figured out that nginx package couldn't be installed because the nginx-common dependancy isn't installed.
Going up in the docker compose build output there was another error. The error showed that the installation of nginx-common failed because it was waiting for the user input about what to do with the already existing /etc/nginx/nginx.conf file.
Checking the Dockerfile of the proxy, I found out that the installing of nginx comes after the copy of the /etc/nginx/nginx.conf file.

### Root cause
Installing error because user input was needed. User input was needed because nginx.conf file already copied to /etc/nginx before installing nginx.

### Fix
Changed the Dockerfile to install nginx first, and then copy the nginx configuration file to the right path.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
proxy-1        | 2026/10/18 10:33:27 [error] 7#7: *2 connect() failed (111: Connection refused) while connecting to upstream, client: 172.18.0.4, server: localhost, request: "GET /api/info HTTP/1.1", upstream: "http://127.0.0.1:8080/api/info", host: "proxy:80"

### How I diagnosed it
I saw using the error message that the proxy couldn't access the backed.
I looked at docker-compose.yml to see what is the open ports for the proxy and backend and the service names.
Everything looked valid. I checked nginx.conf to see what is the configured ip and port for the backend service.
The upstream ip was 127.0.0.1 (with the right port) which wouldn't work for accessing another container.

### Root cause
Upstream ip address was configured wrong for the proxy on nginx.conf

### Fix
changed from localhost address to "backend" the service name full the optimal container network access.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
test-runner-1  | TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway
proxy-1        | 2026/10/18 11:26:58 [error] 7#7: *3 connect() failed (111: Connection refused) while connecting to upstream, client: 172.18.0.4, server: localhost, request: "GET /api/info HTTP/1.1", upstream: "http://172.18.0.2:8080/api/info", host: "proxy:80"

### How I diagnosed it
I saw that it still got connection refused from the backend although the container networking configuration looked fine.
I tried to run independently docker compose up to inspect the backend and proxy container problems.
I saw that I get the connection refused error from test-runner before I get "Backend listening on port 8080..." message from the backend.
Looked again at the docker-compose.yml and saw dependancies that made sense.
Advised with AI for how can I make test-runner wait until the backend starts listening and it suggested to add healthchecks to make the dependent services wait until the backend /health returns code 200 (instead for just waiting for the container to start).
Entered: https://docs.docker.com/compose/how-tos/startup-order/ to make sure the AI suggestion make sense and is supported by docker.
After I configured the healthcheck I got another problem that I saw that the backend was listening but still the healthchecks fail.
I Asked the AI for a command so I can see the logs from the healthcheks.
Then I used:  docker inspect --format "{{json .State.Health}}" and saw that the problem was that alpine image doesn't have curl command for the healthcheck.
With the help from AI for the right syntax, I replaces the curl with wget.

### Root cause
test-runner started running before the backend was started listening and get requests.

### Fix, and why I chose it over the alternatives
I added healthcheck in docker-compose.yml for the backend so it will be considered healthy just when \health will return successfully.
I added healthcheck for the proxy service to wait until /nginx-health is successful for best practice.
Then I configured the dependancies to wait until the services are healthy.
I chose this over other alternatives (like just making test-runner wait for few seconds) in order to keep the dependancy mechanism clear and optimal. In this way, we configure accurately what is considered healthy for the service and what should be the dependancies.

---

## Defect 4: CI pipeline fails

### Symptom / error output
(I am not sure if this is the intended error because the pipeline was successful when I pushed my changes, I answer according to the error in the initial commit failed pipeline that I solved).
line 1: ./scripts/run-tests.sh: Permission denied

### How I diagnosed it
I ran ls -l on the scripts folder and saw that indeed run-tests.sh missed the execute permissions.

### Root cause
No execute permissions for run-tests.sh

### Fix
I added the neccessary permission by running chmod u+x scripts/run-tests.sh

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.
1. Reducing the proxy image size:
initial size: 443MB.
I removed the unnessary package installment of git, htop and vim.
This by itself made the image size: 223MB.
Searched for a minimal base image that would work best for our nginx server.
With the help of AI I decided for nginx:alpine as base image because it's official image and most convenient yet minimal.
I removed the installation commands totally.
Now the image size is: 102MB.
It is still not that minimal and I wanted to try to make it smaller.
I found this base image that contain no shell at all so it is by far minimal: cgr.dev/chainguard/nginx:latest
I had to adapt nginx.conf and docker-compose.yml to switch to port 8080 instead of port 80.
Also for the healthcheck, I decided it is important enough to include wget although the extra space and switched for wget healthcheck instead of curl.
Final image size: 31.3MB

2. Making the proxy run as non-root user:
Fortunatly, The minimal base image I chose is already running as non-root user.
All I was left to do is to make sure that nginx is indeed ran as non-root user.
I ran docker top and the uid of the process user was a non-root user as wanted.

3. Additional improvements:
The base image I chose is very secure because it doesn't have shell and run as non-root user.
The last imporvement I made (not for the proxy image though) is to delete the publically open port 8080 for the backend so it won't be accessible from the public network and just the proxy could access it.
>

---

## Anything you would do differently with more time
In the last part, I wanted to optimize the proxy image for being production ready and change the base image tag to some constant stable tag.
But then I found out that chainguard from which I used their image open only latest for free public use.
If I had more time I would research more about the possible base images to be able to use a non latest tag and preferably something from Dockerhub instead.
