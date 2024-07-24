# AWS VPC, Subnets, Security Groups, and EC2 Instances Setup Script

Цей скрипт для створення VPC, публічної та приватної підмереж, Security Groups та EC2 інстанцій на AWS за допомогою AWS CLI.

## Вимоги

- Встановлений [AWS CLI](https://aws.amazon.com/cli/)
- Встановлений [jq](https://stedolan.github.io/jq/)
- Встановлений [base64](https://www.gnu.org/software/coreutils/base64)

## Використання



```bash

# ID's
AMI_ID
VPC_ID
IGW_ID
SUBNET_PUBLIC_ID
SUBNET_PRIVATE_ID
RT_ID
SG_FRONT_ID
SG_BACK_ID
DB_INSTANCE_ID
WEB_INSTANCE_ID
WEB_PUBLIC_IP
DB_PRIVATE_IP

# Функція для збереження виводу команди у JSON файл
save_output() {
  local command_name=$1
  local output=$2
  echo "$output" > "${command_name}.json"
}

# Створити VPC
VPC_OUTPUT=$(aws ec2 create-vpc --cidr-block 192.168.0.0/24 --output json)
save_output "create_vpc" "$VPC_OUTPUT"
VPC_ID=$(echo "$VPC_OUTPUT" | jq -r '.Vpc.VpcId')

# Створити інтернет-шлюз і прикріпити його до VPC
IGW_OUTPUT=$(aws ec2 create-internet-gateway --output json)
save_output "create_internet_gateway" "$IGW_OUTPUT"
IGW_ID=$(echo "$IGW_OUTPUT" | jq -r '.InternetGateway.InternetGatewayId')
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Створити публічну підмережу
SUBNET_PUBLIC_OUTPUT=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 192.168.0.0/26 --output json)
save_output "create_subnet_public" "$SUBNET_PUBLIC_OUTPUT"
SUBNET_PUBLIC_ID=$(echo "$SUBNET_PUBLIC_OUTPUT" | jq -r '.Subnet.SubnetId')
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_PUBLIC_ID --map-public-ip-on-launch

# Створити приватну підмережу
SUBNET_PRIVATE_OUTPUT=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 192.168.0.64/26 --output json)
save_output "create_subnet_private" "$SUBNET_PRIVATE_OUTPUT"
SUBNET_PRIVATE_ID=$(echo "$SUBNET_PRIVATE_OUTPUT" | jq -r '.Subnet.SubnetId')

# Створити таблицю маршрутизації і додати маршрут для інтернет-шлюзу
RT_OUTPUT=$(aws ec2 create-route-table --vpc-id $VPC_ID --output json)
save_output "create_route_table" "$RT_OUTPUT"
RT_ID=$(echo "$RT_OUTPUT" | jq -r '.RouteTable.RouteTableId')
aws ec2 create-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --subnet-id $SUBNET_PUBLIC_ID --route-table-id $RT_ID

# Створити дві Security Group
# Front
SG_FRONT_OUTPUT=$(aws ec2 create-security-group --group-name sg-FRONT --description "Front security group" --vpc-id $VPC_ID --output json)
save_output "create_sg_front" "$SG_FRONT_OUTPUT"
SG_FRONT_ID=$(echo "$SG_FRONT_OUTPUT" | jq -r '.GroupId')
aws ec2 authorize-security-group-ingress --group-id $SG_FRONT_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
# Back
SG_BACK_OUTPUT=$(aws ec2 create-security-group --group-name sg-BACK --description "Back security group" --vpc-id $VPC_ID --output json)
save_output "create_sg_back" "$SG_BACK_OUTPUT"
SG_BACK_ID=$(echo "$SG_BACK_OUTPUT" | jq -r '.GroupId')
aws ec2 authorize-security-group-ingress --group-id $SG_BACK_ID --protocol tcp --port 22 --source-group $SG_FRONT_ID

# Створити дві t2.micro віртуалки
KEY_PAIR_OUTPUT=$(aws ec2 create-key-pair --key-name my-key-pair --output json)
save_output "create_key_pair" "$KEY_PAIR_OUTPUT"
KEY_MATERIAL=$(echo "$KEY_PAIR_OUTPUT" | jq -r '.KeyMaterial' | base64)
echo "$KEY_MATERIAL" | base64 --decode > my-key-pair.pem
chmod 400 my-key-pair.pem

# Створити WEB сервер у публічній підмережі
WEB_INSTANCE_OUTPUT=$(aws ec2 run-instances --image-id $AMI_ID --count 1 --instance-type t2.micro --key-name my-key-pair --security-group-ids $SG_FRONT_ID --subnet-id $SUBNET_PUBLIC_ID --output json)
save_output "run_instance_web" "$WEB_INSTANCE_OUTPUT"
WEB_INSTANCE_ID=$(echo "$WEB_INSTANCE_OUTPUT" | jq -r '.Instances[0].InstanceId')

# Створити DB сервер у приватній підмережі
DB_INSTANCE_OUTPUT=$(aws ec2 run-instances --image-id $AMI_ID --count 1 --instance-type t2.micro --key-name my-key-pair --security-group-ids $SG_BACK_ID --subnet-id $SUBNET_PRIVATE_ID --output json)
save_output "run_instance_db" "$DB_INSTANCE_OUTPUT"
DB_INSTANCE_ID=$(echo "$DB_INSTANCE_OUTPUT" | jq -r '.Instances[0].InstanceId')

# Налаштування доступу по ssh
# Дізнатись IP адреси
WEB_PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $WEB_INSTANCE_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
DB_PRIVATE_IP=$(aws ec2 describe-instances --instance-ids $DB_INSTANCE_ID --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

# Додати налаштування в ~/.ssh/config
echo "
Host web-host
  HostName $WEB_PUBLIC_IP
  User ec2-user
  IdentityFile ~/.ssh/my-key-pair.pem

Host db-host
  HostName $DB_PRIVATE_IP
  User ec2-user
  IdentityFile ~/.ssh/my-key-pair.pem
  ProxyJump web-host
" >> ~/.ssh/config
