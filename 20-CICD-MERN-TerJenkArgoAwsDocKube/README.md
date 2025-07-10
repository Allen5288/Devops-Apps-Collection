# End to End devops projects

## 1. Operation Process

<https://blog.stackademic.com/advanced-end-to-end-devsecops-kubernetes-three-tier-project-using-aws-eks-argocd-prometheus-fbbfdb956d1a>

Config the EC2
1. create aws ec2 with x2large cause we need larges of things to install
2. create iam role with ec2 admin for test
3. create security group for opening port as 8080 9000 9090
4. install scripts for ec2 can find in 20-CICD-MERN-TerJenkArgoAwsDocKube\Kubernetes-Jenkens-DevSecOps-Project\Jenkins-Server-TF\tools-install.sh
5. session manager to connect ec2
6. sudo su ubuntu
7. sudo htop can see what install the histroy

Config Jenkin
8. <http://54.89.183.209:8080/login> replease with your aws ip address for jenkin login
9.  password can be use  systemctl status jenkins.service in seesiuon manager to see
10. install plugin AWS CredentialsVersion, pipeline: AWS StepsVersion, Terraform
11. credential - add your aws role admin access key and screct
12. Tools - add terraform, use whereis terramform on cli to see the location
13. create job, select pipeline


