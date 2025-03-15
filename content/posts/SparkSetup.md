---
title: "tientran0826"
date: 2025-03-15
draft: false
tags: ["spark", "hadoop"]
categories: ["Tutorial"]
title: 'Cài đặt và setup cụm Apache Spark trên Hadoop 3.3 với VirtualBox'
---

![Demo](https://cdn.discordapp.com/attachments/1149575593403371574/1350496012153786439/image.png?ex=67d6f2ec&is=67d5a16c&hm=6f4dc3edc5d81cafa8871638558e59ecae857aebdc320b12018cc2a6afcf4a99&)

## 1. Chuẩn bị môi trường ảo hoá
- Ảo hoá máy tạo các VMs để giả lập các cụm máy chủ
- Tool ảo hoá: Oracle Virtual Box  (https://www.virtualbox.org/wiki/Downloads)
- Ubuntu 20.04.6 LTS (https://releases.ubuntu.com/focal/)
### 1.1 Setup máy ảo gốc
- Máy này mình cho làm Master, cài đủ các package sau đó Clone ra các máy phụ sau =)))
- Thao tác trực tiếp trên Virtual Box (provip có thể sử dụng Vagrant để auto)
- Lưu ý khi tạo máy ảo -> Setting apdapter mạng thành **Host-only** hoặc **Bridge** (ko khuyến khích Bridge vì ngồi cấu hình IP hồi rớt mạng ráng chịu)
![Network-Apdapter](https://cdn.discordapp.com/attachments/1149575593403371574/1350497733345280071/image.png?ex=67d6f486&is=67d5a306&hm=9d5fd526ea0b806e71f99bafdc35b374efef601ed2257b92959f4d84d88de5c3&)
- Kiểm tra IP thông với máy host

    ```bash
    ip -4 addr show | grep inet
    ```
![your-ip](https://cdn.discordapp.com/attachments/1149575593403371574/1350501992694419586/image.png?ex=67d6f87e&is=67d5a6fe&hm=21f09eed0e45359d45c624522c8536f84e594e0330baa8f26681d620e146b3ed&)
Ở đây địa chỉ được thông với máy host là *192.168.56.102* - Có cùng dãy IP với máy Host (nếu máy host thì kiểm tra trong /ipconfig) hoặc ping trực tiếp từ máy Host đến IP trên.
![ping-vm-ip](https://cdn.discordapp.com/attachments/1149575593403371574/1350503191673831476/image.png?ex=67d6f99c&is=67d5a81c&hm=a6bad5ed0970d4060779d13a2d97fb36a545f43a1c320be7b265ec2dd76dab47&)
## 1.2 Cài đặt một số package chung
### Java
Lệnh cài đặt
```bash
sudo apt-get update
sudo apt-get install openjdk-11-jdk
```
Kiểm tra cài đặt
```bash
java -version
```
Kiểm tra path java
```bash
readlink -f $(which java)
```

![java-version](https://cdn.discordapp.com/attachments/1149575593403371574/1350504961058013257/image.png?ex=67d6fb41&is=67d5a9c1&hm=f9bb71031f28c442f1648e74d04968dd2e726866371e8f568e97ab29741061b1&)

Trong đó đường dẫn */usr/lib/jvm/java-11-openjdk-amd64/bin/java* là đường dẫn HOME_PATH của java -> Nhớ lưu cái này để xíu setup.

### Scala
Cài đặt scala
```bash
sudo apt-get install scala
```
Check version scala
```bash
scala -version
```

### Apache Spark

=))) Phần chính 🤡

Truy cập https://spark.apache.org/downloads.html chọn version phù hợp ở đây mình chọn bản prebuild cho Hadoop 3.3 -> Spark phiên bản 3.5.5
![Spark-download](https://cdn.discordapp.com/attachments/1149575593403371574/1350506302782308436/image.png?ex=67d6fc81&is=67d5ab01&hm=b0cefc35c6647bcdda1089736eb22b5936e2ee5bee5ab267f11542fce975f817&)

Kéo file về bằng wget
```bash
wget https://dlcdn.apache.org/spark/spark-3.5.5/spark-3.5.5-bin-hadoop3.tgz
```
![wget-spark](https://cdn.discordapp.com/attachments/1149575593403371574/1350506928320680098/image.png?ex=67d6fd16&is=67d5ab96&hm=26cdbd720ec98ff75384bc379dbbdba738bb5369c7ab243fa2f8e93ea112941f&)


Giải nén tệp .tgz với *spark-3.5.5-bin-hadoop3.tgz* là tập vừa kéo
```bash
tar xvf spark-3.5.5-bin-hadoop3.tgz
```

Move folder đã giải nén vào path */usr/local/spark*
```bash
sudo mv spark-3.5.5-bin-hadoop3 /usr/local/spark
```

Thêm spark path vào môi trường hệ thống (tương tự set enviroment variables trong Window)
```bash
sudo nano ~/.bashrc
```
Với file *.bashrc* chứa các biến môi trường của hệ thông, thêm vào file
```
export PATH=$PATH:/usr/local/spark/bin
```
Apply biến
```bash
source ~/.bashrc
```
Kiểm tra spark đã được thêm vào môi trường
```bash
which spark-shell
```
![spark-checker](
https://cdn.discordapp.com/attachments/1149575593403371574/1350510702326054952/image.png?ex=67d7009a&is=67d5af1a&hm=a8ed66d84b7b2e395487e38d011cf03408f1fba82b08f739e4591fe0468ca365&)

Vậy ta đã setup một số thư viện cần thiết cho một cụm

## 2. Clone VMs và thiết lập connection giữa các cụm

![clone-vm](https://cdn.discordapp.com/attachments/1149575593403371574/1350512143610679329/image.png?ex=67d701f2&is=67d5b072&hm=ac0afdf1815e00ec0edc64bef698522788f5c463bc319c07431a15c04f21593d&)

Clone VM gốc đã được tạo ở trên thành các worker, số lượng bao nhiêu thì tuỳ theo options người dùng. Ở đây mình clone thêm 2 bản đặt tên là Slave1 và Slave2 và cụm được setup ban đầu sẽ là Master.

### 2.1 Set lại hostname (tuỳ chọn)
- Ở đây mình set lại hostname để tiện quản lý =)))
- Với cụm master được hostname là pd-master
- Các cụm worker được hostname là pd-slave-1 và pd-slave-2

Cú pháp chuyển hostname
```bash
sudo nano /hostname
```
```bash
sudo hostnamectl set-hostname pd-master #Tuong tu voi slave-1, slave-2
```
![change-hostname](https://cdn.discordapp.com/attachments/1149575593403371574/1350513872628613172/image.png?ex=67d7038e&is=67d5b20e&hm=7f30cce087060839d065920e6a6dba657846aa96fa42973c378257731569f6bc&)
Reboot để thấy hostname thay đổi
```bash
sudo reboot
```
![hostname](https://cdn.discordapp.com/attachments/1149575593403371574/1350514218356441169/image.png?ex=67d703e0&is=67d5b260&hm=83aa8fc2f1970ad206c0b2035ba47cd60c95984b6dfe4a225aef2c9514d1c24c&)
Với pd-master là hostname mới, ở đây mình đổi từ trước nên các các lệnh ở hướng dẫn trên đều mặc định là pd-master 

### 2.2 Add hosts cho master và slaves

Như cú pháp kiểm tra địa chỉ ip như phần 1 đã hướng dẫn ta lần lượt có các địa chỉ như sau:
```bash
192.168.56.102 pd-master
192.168.56.103 pd-slave-1
192.168.56.104 pd-slave-2
```

Thêm vào file hosts từng VMs để chúng biết địa chỉ phân giải khi ta gọi các máy bằng hostname

```bash
sudo nano /etc/hosts
```
![input-hosts](https://cdn.discordapp.com/attachments/1149575593403371574/1350515672857448448/image.png?ex=67d7053b&is=67d5b3bb&hm=b3109f014695bb8c638fdf04efcbb53f72685e806d8b07d19b019a35c9524db3&)

Save và kiểm tra hosts được thêm bằng cách ping từ một VM bất kỳ đến các VMs còn lại
![ping-hosts](https://cdn.discordapp.com/attachments/1149575593403371574/1350516144330641408/image.png?ex=67d705ac&is=67d5b42c&hm=03717ec8f41658950548c2a150dcb2c1cac7636090721230e29e7f500ce8dcd3&)

## 3. Setup máy Master
### 3.1 SSH đến Slaves
Config SSH cho máy master sử dụng ssh-keygen để đặp nhập thay vì password vì lý do bảo mật + nhanh và tự động hơn.

Cài đặt *Open SSH Server-Client*
```bash
sudo apt-get install openssh-server openssh-client
```
Generate key pairs
```bash
ssh-keygen -t rsa -P ""
```
![keygen-generator](https://miro.medium.com/v2/resize:fit:750/format:webp/1*htA6Tx0mz0ZZEf4k0oKcMg.png)

Đẩy key vừa tạo vào authorized_keys
```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

Đưa ssh-keygen vừa được tạo vào key được truy cập của cả máy master và các máy slaves

```bash
ssh-copy-id user@pd-master #Thay user bang account name
ssh-copy-id user@pd-slave-1
ssh-copy-id user@pd-slave-2
```
Kiểm tra ssh từ máy master đến các máy slaves

```bash
ssh pd-slave-1 # Test ssh den slave 1
ssh pd-slave-2 # Test ssh den slave 2
```

### 3.2 Spark config cho Master
Chuyển đến thư mục config của Spark
```bash
cd /usr/local/spark/conf
```
Copy template config của spark vào cùng thư mục
```bash
cp spark-env.sh.template spark-env.sh
```
Giờ là lúc config =)))
```bash
sudo nano spark-env.sh
```
Thêm đoạn này vào (thay địa chỉ và đường dẫn)
```bash
export SPARK_MASTER_HOST='<MASTER-IP>'
export JAVA_HOME=<Path_of_JAVA_installation>
```
Với MASTER_IP với máy master như ví dụ là *192.168.56.102* và JAVA_HOME là */usr/lib/jvm/java-11-openjdk-amd64/bin/java*.

### 3.3 Thêm workers
Trong cùng thư mục */usr/local/spark/conf*
```bash
sudo nano slaves
```

Thêm nội dung như sau

```bash
pd-master
pd-slave-1
pd-slave-2
```
### 3.4 Khởi động Spark Cluster
Trên máy Master

```bash
cd /usr/local/spark
./sbin/start-all.sh
```
![spark-start](https://cdn.discordapp.com/attachments/1149575593403371574/1350521088278401205/image.png?ex=67d70a46&is=67d5b8c6&hm=9e4aa1e69f63c93f6a5226807ee5c0164a16f5b37a8bc77fc81ee551deb09fad&)

Lệnh stop
```bash
./sbin/stop-all.sh
```

## 4. Kiểm tra cụm Spark
### 4.1 Spark UI
Kiểm tra thông qua Spark UI, từ máy host hoặc trên các VMs truy cập

```
http://<MASTER-IP>:8080/
```

![Spark-UI](https://cdn.discordapp.com/attachments/1149575593403371574/1350521799984677045/image.png?ex=67d70af0&is=67d5b970&hm=7de39c71b31393fb3606bf7334c7b4619c26d51d4be42c676e750b34e49d3085&)

### 4.2 Check qua code
- Lười cài JAVA trên máy host quá nên mình test code trên máy Master luôn.
- Note: Nhớ cài pyspark =)))

```bash
nano test_spark.py
```

Code demo:
```python
from pyspark.sql import SparkSession
import os

spark = SparkSession.builder\
            .appName("Check Cluster Spark)\
            .master("spark://<MASTER-IP>:7077)\
            .getOrCreate()
data = [("Alice",25)]
df = spark.createDataFrame(data, ["Name", "Age"])
df.show()
# Them cai nay de hoi chay file py trong command khong bi end session ngay lap tuc
input("Inter to exit") 
```
Kết quả:
![Demo](https://cdn.discordapp.com/attachments/1149575593403371574/1350496012153786439/image.png?ex=67d6f2ec&is=67d5a16c&hm=6f4dc3edc5d81cafa8871638558e59ecae857aebdc320b12018cc2a6afcf4a99&)
