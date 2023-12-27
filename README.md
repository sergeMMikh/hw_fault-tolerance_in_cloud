# Домашнее задание к занятию «Отказоустойчивость в облаке» - Михалёв Сергей
---

##  Задание 1 

### В виду проблем с доступом к Vandex из-за границы, использую для выполнения данного домашнего задания сервис AWS Amazon.

Возьмите за основу [решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке»](https://github.com/netology-code/sdvps-homeworks/blob/main/7-03.md#задание-1).

1. Теперь вместо одной виртуальной машины сделайте terraform playbook, который:

- создаст 2 идентичные виртуальные машины. Используйте аргумент [count](https://www.terraform.io/docs/language/meta-arguments/count.html) для создания таких ресурсов;
- создаст [таргет-группу](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_target_group). Поместите в неё созданные на шаге 1 виртуальные машины;
- создаст [сетевой балансировщик нагрузки](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer), который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.

Рекомендуем изучить [документацию сетевого балансировщика нагрузки](https://cloud.yandex.ru/docs/network-load-balancer/quickstart) для того, чтобы было понятно, что вы сделали.

2. Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

3. Перейдите в веб-консоль Yandex Cloud и убедитесь, что: 

- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

4. Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.

## Результат

*1. Terraform Playbook.*
```
resource "aws_key_pair" "laptop_ssh" {
    key_name = "smm@smm-X550LN"
    public_key = file("~/.ssh/id_rsa.pub")
}

resource "aws_instance" "test_ubuntu_nginx" {
    count = 2
    ami = "ami-0a9e7160cebfd8c12"
    instance_type = "t3.micro"
    key_name      = aws_key_pair.laptop_ssh.key_name
    vpc_security_group_ids = [
        aws_security_group.demo_sg.id, 
    ]
    user_data              = file("userdata.tpl")

    tags = {
      Name = "ubuntu-4-hw"
      Owner = "SMMikh"
    }
}


resource "aws_lb_target_group" "hw_tg" {
  name     = "tf-example-lb-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = "${aws_default_vpc.default.id}"
}


resource "aws_lb_target_group_attachment" "lbtargetgroupatt" {
  # covert a list of instance objects to a map with instance ID as the key, and an instance
  # object as the value.

  
  for_each = {
    for k, v in aws_instance.test_ubuntu_nginx:
    v.id => v
  }

  target_group_arn = aws_lb_target_group.hw_tg.arn
  target_id        = each.value.id
  port             = 80
}

# Default VPC
resource "aws_default_vpc" "default" {
}

# Security group
resource "aws_security_group" "demo_sg" {
  name        = "demo_sg"
  description = "allow ssh on 22 & http on port 80"
  vpc_id      = aws_default_vpc.default.id
  # vpc_id      = "${aws_vpc.main.id}"

  ingress {
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  ingress {
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }
}

# DATA
data "aws_ami" "aws_ubuntu" {
  most_recent = true
  owners      = ["amazon"]

 filter {
    name   = "name"
    values = ["amzn-ami-hvm*"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
} 
```

*2. Скриншот статуса балансировщика и целевой группы.*
* <img src="https://drive.google.com/file/d/1V4EBiD0VdRJJp17JjNTpLVvwFwKvl_-0/view?usp=sharing" alt="Task_2_1" width="500" height="auto">
*2.1 Описания полученных aws instanses*
* <img href="https://drive.google.com/file/d/1VFoDSsGPMUGdtCPTIUZ6wTSt-pavD65Y/view?usp=sharing" alt="Task_2_1" width="500" height="auto">
* <img href="https://drive.google.com/file/d/1VPVwq0pG_UA3YmjmTnBtmyrwjjgiZraG/view?usp=sharing" alt="Task_2_1" width="500" height="auto">
*2.2 Описание балансировщика*
* <img href="https://drive.google.com/file/d/1VWdj_SNvop6W42VaCNPDLaQTIXKs1cDL/view?usp=sharing" alt="Task_2_1" width="500" height="auto">

*3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*
* <img href="https://drive.google.com/file/d/1VWxT1ybVyk1ty71HU9sIHwWI1MFNPInG/view?usp=sharing" alt="Task_2_1" width="500" height="auto">
* <img href="https://drive.google.com/file/d/1VbeYEl8f1UEDY9-4ybuZEgllu-s59Uj7/view?usp=sharing" alt="Task_2_1" width="500" height="auto">

---
