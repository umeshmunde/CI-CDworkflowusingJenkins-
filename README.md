# Jenkins CI/CD + Terraform (IaC) Project 

**Jenkins pipeline** (checkout → build → test → docker build → deploy → verify) and a **Terraform configuration** (reusable module that provisions an EC2 server + security group).

```
cicd-project/
├── Jenkinsfile                      <- Declarative pipeline (submit this)
├── app/
│   ├── server.js                    <- Node.js/Express sample app
│   ├── package.json
│   ├── test/app.test.js             <- Jest tests (the "Test" stage)
│   ├── Dockerfile                   <- Build/deploy artifact
│   └── .dockerignore
├── terraform/                       <- IaC (submit this)
│   ├── versions.tf  main.tf  variables.tf  outputs.tf
│   ├── terraform.tfvars.example
│   └── modules/web_server/          <- reusable module (EC2 + SG)
├── scripts/
│   ├── setup-ec2-jenkins.sh         <- installs Jenkins, Docker, Node, Terraform
│   └── collect-evidence.sh          <- gathers proof for screenshots
└── .gitignore
```

---

## PART 1 — Launch the EC2 server (AWS Console)

| Setting | Value |
|---|---|
| AMI | Ubuntu Server 22.04 LTS |
| Instance type | **t3.small** (or t2.small) — t2/t3.micro is too small for Jenkins |
| Storage | 20 GB gp3 |
| Key pair | create/download a `.pem` |
| Security group inbound | **22** (SSH, My IP), **8080** (Jenkins, My IP), **3000** (App, My IP or Anywhere) |

## PART 2 — Install everything on the server

```bash
# From your laptop: copy the zip up and connect
scp -i your-key.pem cicd-project.zip ubuntu@<EC2-PUBLIC-IP>:~
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>

# On the server
sudo apt-get update && sudo apt-get install -y unzip
unzip cicd-project.zip && cd cicd-project
sudo bash scripts/setup-ec2-jenkins.sh
```

The script prints the Jenkins admin password at the end. 📸 **Screenshot #1:** the tool versions printed by the script.



## PART 2 — Configure Jenkins

1. Open `http://<EC2-PUBLIC-IP>:8080`, paste the admin password, choose **Install suggested plugins**, create an admin user.
2. **New Item** → name `cicd-demo-pipeline` → **Pipeline** → OK.
3. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: your GitHub URL
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
4. Save.


## PART 3 — Run the pipeline (CI/CD Execution)

Click **Build with Parameters** → leave defaults (`RUN_TERRAFORM_PLAN` ✔, `TERRAFORM_APPLY` ✘) → **Build**.

Stages that run: Checkout → Install deps → Test → Docker build → Terraform validate → Deploy → Verify.

📸 **Screenshot #4:** Stage View, all green.
📸 **Screenshot #5:** Console Output ending with `Finished: SUCCESS`.
📸 **Screenshot #6:** browser at `http://<EC2-PUBLIC-IP>:3000/` showing the JSON response.

Then run `bash scripts/collect-evidence.sh` and screenshot the output (📸 **#7**).

**Optional – show it's truly automated:** edit the message in `app/server.js`, `git commit && git push`, click Build again, refresh the browser — the change is live. (Bonus: in the job config enable *Poll SCM* with `* * * * *` so pushes trigger builds automatically.)

## PART 4 — Terraform (Infrastructure as Code)

Terraform is already installed on the server. Provide AWS credentials (easiest for a demo: an IAM user with EC2 permissions):

```bash
aws configure          # enter Access Key, Secret, region ap-south-1
cd ~/cicd-project/terraform
cp terraform.tfvars.example terraform.tfvars

terraform init
terraform fmt -recursive
terraform validate
terraform plan         # 📸 Screenshot #8: plan showing 2 resources to add
terraform apply        # type: yes   📸 Screenshot #9: "Apply complete! Resources: 2 added" + outputs
```

Open the `app_url` output in a browser (nginx welcome page) — 📸 **#10**. Check the AWS Console → EC2 to see the new instance — 📸 **#11**.

**When finished, destroy to avoid charges:**
```bash
terraform destroy      # type: yes
```

> Optional: tick `TERRAFORM_APPLY` in Jenkins to run apply from the pipeline. That needs AWS credentials available to the `jenkins` user (`sudo -u jenkins aws configure`). The safer demo is plan/validate in Jenkins and apply manually.

---

