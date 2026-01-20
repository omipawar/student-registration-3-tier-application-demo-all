## Setup project infrastructure using terraform modules

**launch and ec2 instance**
- instance type should be medium or large
- 30 gb disk
**create iam role**
  - give admin acess
  - and attach to ec2 instance
**Clone Repository**
````
git clone https://github.com/abhipraydhoble/student-registration.git
````
````
cd student-registration/infra
````


**Terraform Installation:Ubuntu**
````
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y
````


#### Changes:
- **main.tf** instance-type
- **provider.tf** add provider details 

**Initialize terraform**
````
terraform init
````
---
**AWS CLI Installation:**

````
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
````
````
aws --version
````
**Cluster Login**
- first login into aws
````
aws configure
````
````
aws eks update-kubeconfig --name cbz-cluster
````
---
**:Install kubectl**

````
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
````
````
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
````
````
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
````
````
kubectl version --client
````



### 1. Setup MariaDB 
- wait for cluster and rds creation
- connect to cluster worker node

```bash
sudo yum update -y
sudo yum install mariadb105-server -y
```
**Login To RDS**
````
mysql -h <rds-endpoint> -u <db-username> -p<password>
````

- Create the database:

```sql
CREATE DATABASE student_db;
```
```
USE student_db;
```

- Create the students table:

```sql
CREATE TABLE `students` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `course` varchar(255) DEFAULT NULL,
  `student_class` varchar(255) DEFAULT NULL,
  `percentage` double DEFAULT NULL,
  `branch` varchar(255) DEFAULT NULL,
  `mobile_number` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=80 DEFAULT CHARSET=latin1 COLLATE=latin1_swedish_ci;
```

- Exit MySQL:

```bash
exit
``
- Note: check port no 3306 added in security group



