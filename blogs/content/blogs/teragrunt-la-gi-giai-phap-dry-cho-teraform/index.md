+++
title = 'Terragrunt là gì? Giải pháp DRY cho Terraform, thoát khỏi ác mộng Copy-Paste'
math = true
date = 2024-09-24T02:09:53Z
draft = false
series = ["Terragrunt"]
series_order = 1
+++

## Terragrunt đã cứu chàng Data Engineer năm đó

Cậu sinh viên tên Thành, năm nào giờ đã trở thành một Data Engineer đầy nhiệt huyết. Mới học xong khóa Terraform cơ bản, cậu hăng hái xung phong làm DevOps cho dự án chuyển đổi hệ thống on-premise lên cloud của công ty. Trách nhiệm của cậu? Provision toàn bộ services trên AWS cloud bằng Infrastructure as Code.

**Ban đầu, mọi thứ thật tuyệt…**

Cậu tự tin gõ những dòng code Terraform đầu tiên, mọi thứ chạy mượt mà. Deployment sạch sẽ, không lỗi, mọi người vỗ tay khen ngợi.

**Nhưng rồi... ác mộng ập đến.**

Dự án ngày càng phình to, gần 20 cái S3 bucket, chục cái job Glue, hàng trăm cái Lambda functions, vài chục role IAM, vài con database RDS, chưa kể subnet, security group nhiều vô kể. Nhìn lại codebase Terraform của mình và… tim như muốn rớt ra ngoài.

- Hàng nghìn dòng copy-paste giống hệt nhau, chỉ khác tên và một vài thuộc tính setting.
- Một thay đổi nhỏ liên quan đến phần base? Phải sửa ở nhiều chỗ. Quên sửa 1 chỗ là sai.
- Bảo trì, debug, thêm tính năng mới? Mất cả tiếng đồng hồ để tìm chỗ cần sửa.

Lúc này, cậu tự hỏi: "Chẳng lẽ Terraform là một cú lừa?"
Vừa bực bội vừa hoang mang, cậu lao đầu đi tìm hiểu từ google, chatgpt, youtube, và rồi cậu gặp [Terragrunt](https://terragrunt.gruntwork.io/). Click vào đường link, đập vào mắt cậu là dòng chữ `DRY and maintainable OpenTofu/Terraform code.`

**Vậy là cậu lao vào nghiên cứu. Và rồi… mọi thứ thay đổi.**

- ✔️ Không còn copy-paste hàng trăm lần để tạo các services tương tự nhau.
- ✔️ Quản lý multi-environment dễ dàng, nhất quán hơn.
- ✔️ Terraform modules trở nên gọn gàng, không bừa bộn.
- ✔️ Quan trọng nhất: KHÔNG CÒN OT!

Sau đây, là những gì cậu đã học được - những kinh nghiệm xương máu giúp bạn deploy infra một cách dễ dàng hơn với Terragrunt, cuối serial bài viết này là một demo về cách sử dụng terragrut trong môi trường doanh nghiệp lớn.

## Terragrunt là gì ?

Terragrunt là một wapper của Terraform, giúp bạn quản lý Terraform codebase một cách dễ dàng hơn. Terragrunt giúp bạn giải quyết những vấn đề phổ biến của Terraform:

- **DRY (Don't Repeat Yourself)**: Không lặp lại code, giúp codebase gọn gàng, dễ bảo trì.
- **Multi-environment**: Quản lý nhiều môi trường (dev, staging, prod) dễ dàng.
- **Remote state**: Quản lý terraform state file trên cloud (S3, GCS, Azure Blob Storage) một cách đơn giản hơn.

## Install

### Install Terraform

Do Terragrunt là một wapper của Terraform. Để sử dụng Terragrunt, bạn cần cài đặt Terraform trước. Bạn có thể cài Terraform thông qua [link này](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli).

Với Windows, bạn có thể cài Terraform thông qua [chocolatey](https://chocolatey.org/):
Nếu chưa có chocolatey, bạn có thể cài đặt bằng cách tham khảo [link này](https://chocolatey.org/install#individual).

```powershell
choco install terraform
```

Verify Terraform đã cài đặt thành công:

```powershell
terraform --version
```

Check được version như sau là thành công.

![alt text](install_terraform.png)

### Install Terragrunt

Có lưu ý là version terragrunt trên chocolatey không phải là bản mới nhất, bạn có thể tải bản mới nhất từ [trang chủ](https://terragrunt.gruntwork.io/docs/getting-started/install/).

Sau khi tải về, giải nén và add vào PATH, chạy lệnh sau để kiểm tra terragrunt đã cài đặt thành công:

```powershell
terragrunt --version
```

![alt text](install_terragrunt.png)

## Cấu trúc thư mục ([Best practice](https://docs.gruntwork.io/2.0/docs/overview/concepts/infrastructure-live))

Đây là cấu trúc thư mục được khuyến nghị khi sử dụng Terragrunt, về cơ bản dự án của mình cũng follow theo phương án tổ chức này:

```bash
account(1.)
 └ _global(2.1)
 └ region(2.2)
    └ _global(3.1)
    └ environment(3.2)
       └ category(4)
          └ resource(5)
```

### 1. Accounts

Root level là `account`, đại diện cho 1 account AWS, thông thường đa phần các tổ chức sẽ deploy non-prod(dev, stage...) trên cùng 1 account AWS, và prod trên account riêng.

### 2.1. _global

_global folder để chứa các config chung cho các services không chia region như IAM role, Route53, CloudTrail...

### 2.2. Regions

Folder `region` chứa file config cho từng region mà bạn sử dụng. Ví dụ, `ap-southeast-1` và `us-west-2`.
Một số tổ chức bỏ qua folder `region` vì họ chỉ sử dụng 1 region duy nhất, nhưng mình khuyến nghị nên tạo folder `region` vì sau này biết đâu sẽ có region Việt Nam, hoặc chuyển về region Thái Lan 🤣

### 3.1. _global(across environments)

_global(across environments) chứa các config chung cho các services có thể dùng chung cho môi trường dev, staging... như SNS, ECR...
Cái này cũng tùy tổ chức muốn dùng chung các services gì, ví dụ có tổ chức non-prod(dev, stage) dùng chung 1 database cho tiết kiệm chi phí.

### 3.2. Environments

Sub-folder `environment` chứa các config cho từng môi trường (dev, staging, prod). Mỗi môi trường sẽ có các config riêng như cấu hình ec2, rds, s3, vpc, subnet, security group, etc. nên sẽ tách riêng cho từng môi trường.

### 4. Categories

Sub-folder `category` dùng để gom nhóm các resources cùng loại với nhau như:

- Networking: VPC, Subnet, Route53, Security Group...
- Compute: EC2, ECS...
- Storage: S3, EBS...

Cũng tùy team, team mình không chia category.

### 5 Resources

Cuối cùng, sub-folder `resource` chứa các resources cụ thể như S3 bucket, IAM role, Glue job, Lambda function...

## Ví dụ cụ thể

Dưới đây là ví dụ cụ thể triển khai 1 account AWS với:

- 1 region `ap-southeast-1`
- môi trường non-prod bao gồm `dev`, `stage`
- môi trường prod
- sử dụng 3 services `s3`, `glue`, `iam`

```bash
non-prod
 └ ap-southeast-1
    └ dev
       └ s3
            └ bucket_data
            └ bucket_logs
       └ glue
            └ job_1
            └ job_2
       └ iam
          └ role
    └ stage
       └ s3
            └ bucket_data
            └ bucket_logs
       └ glue
            └ job_1
            └ job_2
       └ iam
          └ role
prod
 └ ap-southeast-1
    └ prod
       └ s3
            └ bucket_data
            └ bucket_logs
       └ glue
            └ job_1
            └ job_2
       └ iam
          └ role
```

