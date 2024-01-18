## GitLab Offline Security Scanning Setup

### Overview

GitLab scanners usually connect to the internet to download the latest sets of signatures, rules, and patches. A few extra steps are necessary to configure the tools to function properly on an offline environment. 

A Static Application Security Testing (SAST) and Secret Detection will be performed on all the code in the GitLab; they are responsible for maintaining code quality by checking for known vulnerabilities and exposed secrets. 

The process for making these images available without direct access to the public internet involves downloading the images then packaging and transferring them to the offline host. Here’s an example of such a transfer:

1. Download the container images from public internet.
2. Package the container images as tar archives.
3. Transfer the container to offline environment.
4. Load transferred images into offline Docker registry.

The steps are explained in detail below.

This process ensures that offline security scanning images are securely transferred from an online environment (where they were downloaded and prepared) into an offline environment utilizing secure file transfer protocols and intermediate hosts for added security measures.

![Untitled Diagram drawio (2)](https://github.com/nijdarshan/dummy/assets/51939272/dd60a97f-8145-42e5-8aca-ab1fa3a2dea2)

### Step 1: Downloading Images
- **Device Pre-requisites:** Internet connection and Docker/Podman installed.
- **Action:** 
    - Use a personal laptop to download offline security scanning images from GitLab.com from a personal account.
    - Use the below template to download the images:
    ```
      include:
          - template: Security/Secure-Binaries.gitlab-ci.yml
    ```
    More details on [GitLab Documentation](https://docs.gitlab.com/ee/user/application_security/offline_deployments/#using-the-official-gitlab-template)
    - Docker Command: `docker pull <image_name>`
    - Podman Command: `podman pull <image_name>`
    - Zip the images to prepare them for transfer.
    - Docker Command: `docker save -o <path_for_created_tar_file> <image_name>`
    - Podman Command: `podman save -o <path_for_created_tar_file> <image_name>`

### Step 2: Transfer Files to Corporate Laptop
Trasnsfer the zipped images to Corporate Laptop
- **Action:** 
    - Transfer the downloaded images from the personal laptop to the corporate laptop using a thumb drive.

### Step 3: Transfer to Jumphost
- **Environment:** Transition from Online to Offline
- **Action:**
    - Use FTP on the corporate laptop to transfer the zipped file containing renamed images to Jumphost.
    - Ensure secure data transfer protocols are followed.

### Step 4: Transfer to Server Connected Container Registry
- **Environment:** Offline
- **Action:**
    - From Jumphost, use FTP again to transfer zipped files into a server that is connected with container registry.

### Step 5: Unzip, Rename and Push Images
- **Environment:** Offline 
- **Action:**
    - Unzip the transferred files in this server.
    - Docker Command: `docker load -i <path_to_tar_file>`
    - Podman Command: `podman load -i <path_to_tar_file>`
    - Rename the images to add the offline container registry path.
    - Docker Command: `docker tag <old_image_name> <new_image_name>`
    - Podman Command: `podman tag <old_image_name> <new_image_name>`
    - Push theseimages into the offline container registry.
    - Docker Command: `docker push <new_image_name>`
    - Podman Command: `podman push <new_image_name>`
 
The below python script can also be used to perform the above action:
```python
import os

entries=os.listdir("./")
for entry in entries:
    if entry.endswith(".tar"):
        print(entry)
        os.system("podman load -i "+entry)
        image_split = entry.split(".")
        image_name = image_split[0][:-2] + ":" + image_split[0][-1]
        old_image_name = <old_image_path> + image_name 
        new_image_name = "harbor" + image_name
        print(old_image_name)
        print(new_image_name)
        os.system("podman image tag "+ old_image_name + " " + new_image_name)
        os.system("podman push "+image_name)
```

Please replace `<image_name>`, `<old_image_name>`, `<old_image_path>`, `<new_image_name>`, `<path_for_created_tar_file>`, and `<path_to_tar_file>` with your actual values. Also, ensure you have the necessary permissions to execute these commands. 


### Custom Security Template

Once the images are transfered, you must tell the offline instance to use these resources instead of the default ones on GitLab.com. To do so, set the CI/CD variable `SECURE_ANALYZERS_PREFIX` with the URL of the project container registry.

You can set this variable in the projects’ .gitlab-ci.yml, or in the GitLab UI at the project or group level. 

Now include the required templates in the custom security template:
```
include:
    template: SAST.gitlab-ci.yml
    template: Secret-Detection.gitlab-ci.yml
```
The security scaning jobs have been added custom script and rules to fail the job if vulnerabilities are found of a certain severity threshold and also to only trigger on explicit pipeline run of any non-main branch of a project. 

Please find the default GitLab security scanning jobs [here](https://gitlab.com/gitlab-org/gitlab/-/tree/master/lib/gitlab/ci/templates/Jobs)

This template is used in the module support functions template which is used in every module. Like below:
```
include:
    local: Security/Security.gitlab-ci.yml
```
Also, make sure to use the `SECURE_BINARIES_ANALYZER_VERSION` tag in every job 
The security jobs run in the test stage of a pipeline before executing the actual run, preventing the unsafe code to run on any of our offline environment.

### Security Policies

Security policeis have also been setup to make sure no unsafe code is merged to the main branch without an approval from the security experts.

A custom scan result policy is a security approval policy that allows approval to be required based on the findings of one or more security scan jobs. Scan result policies are evaluated after a CI scanning job is fully executed and both vulnerability and license type policies are evaluated based on the job artifact reports that are published in the completed pipeline.

![image](https://github.com/nijdarshan/dummy/assets/51939272/4a0036a0-b62d-4621-81bb-5907e4b0c066)


Scan result policies evaluate the artifact reports generated by scanners in your pipelines after the pipeline has completed. Scan result policies focus on evaluating the results and determining approvals based on the scan result findings to identify potential risks, block merge requests, and require approval.

Scan result policies do not extend beyond that scope to reach into artifact files or scanners. Instead, we trust the results from artifact reports. This gives teams flexibility in managing their scan execution and supply chain, and customizing scan results generated in artifact reports (for example, to filter out false positives) if needed.

See more on defining custom security policies [here](https://docs.gitlab.com/ee/user/application_security/policies/scan-result-policies.html)
