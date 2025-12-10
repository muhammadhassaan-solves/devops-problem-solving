--------------------------------------------------------------------------------------------------------------------------------------------------------------
13. ClickJacking and Essential Http Headers
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 09/12/25
Since I was using ALB with AWS Load balancer controller in kubernetes cluster, I explored various methods 
and at the end I found that there is an option in load balancer settings that allow to change attributes of the listener. 
So, I simply turn on these reponse headers as follow:
a. HTTP Strict Transport Security (HSTS) header value
max-age=31536000; includeSubDomains; preload
b. Content Security Policy (CSP) header value
frame-ancestors 'self'
c. X-Content-Type-Options header value
nosniff
d. X-Frame-Options header value
SAMEORIGIN

These response heaeders solved our ClickJacking and Essential Http Headers vulnerabilites of one of 
our critical produciton based application raised by our VAPT Team. 

--------------------------------------------------------------------------------------------------------------------------------------------------------------
12. Pod Internal Connectivity got disturbed after I tried to make some changes in aws native cni for network policy
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 26/11/25
After hours of troubleshooting, I found that changing worker nodes  is the solution. So, I dissociated the worker nodes ec2 from Auto Scaler group one by one and 
the autoscaling group (asg) automatically launched new worker nodes and things started to work properly. 

---------------------------------------------------------------------------------------------------------------------------------------------------------------
11. I was facing 403 error after kubernetes deployment
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 24/11/2025
After hours of troubleshooting, I found that that the client security team added our alb to waf. So, the issue was oversize handling by AWS managed rules on aws.

following ai-assisted search helped me.

a. helm upgrade nginx-ingress ingress-nginx/ingress-nginx -n ingress-nginx \
  --reuse-values \
  --set controller.config.use-forwarded-headers="true" \
  --set controller.config.real-ip-header="X-Forwarded-For" \
  --set controller.config.set-real-ip-from="0.0.0.0/0" #for prod need to specify

b. Change WAF "Oversize Handling" (Most Critical)
You need to tell AWS WAF what to do when the request is larger than it can inspect.
Go to the AWS Console > WAF & Shield > Web ACLs.
Select the Web ACL attached to your ALB.
Go to the Rules tab.
If you are using AWS Managed Rules (like AWSManagedRulesCommonRuleSet or SQLi):
Edit the rule set.
Look for "Oversize handling" configurations.
Set the action to "Continue" (instead of Block/Match) for requests that exceed the inspection limit.
Note: This means WAF inspects the first 8KB, checks for attacks, and if clean, lets the rest of the 800KB pass through blindly.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
10. The client requirement was that they need to inspect east-west traffic inspection inside private subnets where worker nodes are deployed.
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 13/10/2025

Solution:
While launching EKS, I didn't use nat for private subnets. Instead we add TGW in the route tables of private subnets related to worker nodes and it helped the security team to inspect the traffic from worker nodes.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
9. EFS related deployment was not getting successful.
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 12/10/2025

Solution:
After hours of troubleshooting, I found the sg related to nfs need to allow all EC2 instances in the EKS worker node security group to connect via TCP port 2049 to the EFS security group.

Key Learning:
If things related to efs don't work check for logs of efs csi.
for cheking issues, related to networking use aws system manager to the worker nodes and try nslookup with efs dns then use nc to check connectivity. 

--------------------------------------------------------------------------------------------------------------------------------------------------------------
8. Request Size Limitation: Large API Payloads Failing
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 22/09/2025

Problem:
Developers reported that API requests with large payloads (such as file uploads) were failing. The error returned was:
413 Request Entity Too Large
In short, the server was rejecting large requests before they could reach the application backend.

Root Cause:
By default, Nginx limits the maximum allowed request body size to 1 MB.
Any request exceeding this limit was blocked by Nginx, preventing it from being processed by the application.

Solution:
We increased the allowed request size and timeout values in the Nginx configuration.
Under the server block, the following directives were added:
client_max_body_size 200M;
client_body_timeout 900;

After updating the configuration, the changes were tested and applied:
nginx -t
systemctl reload nginx

--------------------------------------------------------------------------------------------------------------------------------------------------------------
7. Github Actions CI/CD Automation: RAG WebUI App Deployment
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 24/09/2025

Problem:
Needed a fully automated CI/CD pipeline to:
Build frontend and backend on push to master.
Create and push Docker image to Docker Hub.
Deploy container to production server.

Root Cause of Failure:
Node.js memory limit: Frontend build required more heap than default.
Peer dependency conflicts: npm ci failed due to unresolved peer deps.

Solution:
Increase Node.js Heap in Docker Build
RUN npm install --legacy-peer-deps
ENV NODE_OPTIONS="--max-old-space-size=8192"
RUN npm run pyodide:fetch && npm run build

--------------------------------------------------------------------------------------------------------------------------------------------------------------
6. CORS Header Configuration Fix
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 1/10/2025

Problem:
Browsers were blocking API requests due to missing or undefined CORS headers.
Error cause:
Access-Control-Allow-Origin: undefined
The backend server attempted to set Access-Control-Allow-Origin, but the value was missing. As a result, browsers rejected the response.

Root Cause:
The .env configuration did not include the allowed frontend domain in the origin list.
When the request domain wasn’t defined, the CORS middleware sent an empty header, triggering browser rejection.

Solution:
Added the required frontend domain to environment variables via Kubernetes secrets.
Ensured Access-Control-Allow-Origin is dynamically set based on allowed origins from environment variables.
Recommended switching to a list-based configuration (e.g., comma-separated domains) instead of hardcoding multiple variables.

DevOps Handling Guide:
If this error reoccurs:
Ask the developer which domain is making the request.
Compare it with the allowed origins in .env.

If missing, update the environment configuration.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
5. Deployment Issue: Updated Code Not Reflecting on Website
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 22/10/2025

Problem:
The developer updated the code, and the GitHub Actions pipeline completed successfully, but the changes were not reflecting on the website.

Root Cause:
When I attempted to manually pull the updated Docker image (instead of relying on the CI/CD pipeline), the image failed to download completely — one of the layers wasn’t pulling successfully. Upon further investigation, I discovered that the instance’s disk space was completely full, which prevented the image from being pulled properly.

Solution:
I ran the following command to clean up unused Docker data and free up space:
docker system prune -f --volumes

This freed up approximately 42 GB of disk space. Afterward, developer was able to pull the image successfully, and the website began reflecting the latest changes.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
4. Application Hang Issue: Node.js Container Not Restarting
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 24/10/2025

Problem:
The Node.js application occasionally became unresponsive when an unhandled error occurred. Although the process was stuck, the container itself continued running, requiring the development team to manually restart it.

Root Cause:
The Dockerfile used by the development team was configured to run the application using nodemon. When an unhandled exception occurred, nodemon did not terminate the container—it kept it in a running state. As a result, neither Docker nor Kubernetes detected the failure, preventing an automatic restart.

Solution:
We updated the setup to use node instead of nodemon in the Dockerfile, ensuring that the container exits when an unhandled error occurs.
Additionally, we added a restart policy in the Docker Compose file to automatically restart containers when they fail.

In the Kubernetes environment, this change aligned perfectly with Kubernetes’ native restart mechanism. When the Node.js process exits, the Pod automatically restarts, ensuring continuous availability.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
3. SSH in WSL2
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 27/10/25

In wsl2, i was facing issue while ssh into ec2 with pem key.
So, following steps will help.
1) You have pem file.
2) use this command to move into /.ssh
mkdir -p ~/.ssh
cp /mnt/c/filelocation ~/.ssh/
3) SSH will reject the key if it’s too open. So set strict permissions:
chmod 600 ~/.ssh/file.pem
4) ssh -i ~/.ssh/file.pem ubuntu@<your-ec2-public-ip>

--------------------------------------------------------------------------------------------------------------------------------------------------------------
2. Frontend Unreachable (502 Bad Gateway)
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 29/10/2025

Problem:
Accessing the application’s frontend via its public URL returned a 502 Bad Gateway error, 
even though the container and NGINX services were running without visible issues.

Root Cause:
The issue occurred because the container’s internal application port did not match the one defined in NGINX. 
(previously, the issue was that app serves on /)

Solution:
Verified which port the application was listening on inside the container:
docker exec -it <container_id> bash -c "apt update -y && apt install -y net-tools lsof > /dev/null 2>&1 && (netstat -tulpn || lsof -i -P -n | grep LISTEN)"
Identified the correct port (8080) and recreated the container with updated port mapping.

--------------------------------------------------------------------------------------------------------------------------------------------------------------
1. The server wasn't working as required. Latency issues, disk issues and memory issues and one time it os got affected.
--------------------------------------------------------------------------------------------------------------------------------------------------------------
Date: 29/10/2025

Problem:
For RAGPOC APP the server wasn't working as required. Latency issues, disk issues and memory issues and one time it os got affected. api request taking too much time

Root Cause:
1 GB availble ram 
containers was crashing
image pull issues
slow requrest
onces it affected the os 

Solution:
Change instance type higher one with more memeory, cpu and networking.
Stop the Instance

Select your instance.
Click Instance state → Stop instance.
Wait until the state becomes stopped.

Change Instance Type
With the instance selected → Click Actions → Instance settings → Change instance type.
From the dropdown, choose a larger instance type (e.g., t3.medium → t3.large or m5.large → m5.xlarge).
Start the Instance

Click Instance state → Start instance.
The instance will boot up with the new size (more CPU, RAM, or network performance).
 
2. Identify Your Volume

Go to EC2 Console → Volumes (under “Elastic Block Store”).
Find the volume attached to your instance (check the “Attachment Information” column).

Modify the Volume

Select your volume.
Click Actions → Modify volume.

Increase the Size (e.g., from 30 GB → 100 GB).
You can also change volume type (e.g., gp2 → gp3 for better performance and lower cost).

Grow the Partition
sudo growpart /dev/nvme0n1 1

Resize the Filesystem
df -hT /

If it shows xfs, then:
sudo xfs_growfs -d /
If it shows ext4, then:
sudo resize2fs /dev/nvme0n1p1
df -h

Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  200G   67G  133G  34% /
