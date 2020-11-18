# Veeva-Workshop-Elasticsearch

## 三、实验操作
### Lab 2、验证AWS ElasticSearch对中文插件的支持。
#### 步骤三：在“Dev Tools”左侧输入下面内容，查看ELasticsearch的分词能力。
```
    POST _analyze 
    {
      "text": "本实验中的一个环节是用来测试服务对中文插件的支持"
    }
```

#### 步骤四：使用”Smart Chinese Analysis“插件，查看分词结果。
```
    POST _analyze 
    {
      "text": "本实验中的一个环节是用来测试服务对中文插件的支持",
      "analyzer": "smartcn"
    }
```
#### 步骤五：使用IK (Chinese) Analysis”，查看分词结果。
```
    POST _analyze 
    {
      "text": "本实验中的一个环节是用来测试服务对中文插件的支持",
      "analyzer": "ik_smart"
    }
        POST _analyze 
    {
      "text": "本实验中的一个环节是用来测试服务对中文插件的支持",
      "analyzer": "ik_max_word"
    }
``` 

### Lab3、EKS集群与应用的日志管理
#### 步骤一：创建IAM Idp，EKS集群会通过该IDP获取IAM Role，并将role与kubernetes service account进行绑定。
```
## 设置环境变量
cat >> ~/.bashrc << EOF
export AWS_REGION = <your-region>
export ACCOUNT_ID = <your-account-id>
export ES_DOMAIN_NAME = <your-elasticsearch-cluster-name>
EOF
source ~/.bashrc
## 绑定iam-oidc-provider
eksctl utils associate-iam-oidc-provider \
    --cluster <your-eks-cluster-name> \
    --approve
```
#### 步骤二：创建AWS IAM Policy。
```
mkdir ~/environment/logging/
cat <<EoF > ~/environment/logging/fluent-bit-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "es:ESHttp*"
            ],
            "Resource": "arn:aws:es:${AWS_REGION}:${ACCOUNT_ID}:domain/${ES_DOMAIN_NAME}",
            "Effect": "Allow"
        }
    ]
}
EoF
aws iam create-policy   \
  --policy-name fluent-bit-policy \
  --policy-document file://~/environment/logging/fluent-bit-policy.json
```
#### 步骤三、创建Kubernetes命名空间与Kubernetes service account，并将IAM policy权限赋予Kubernetes Service Account。
```
## 创建命名空间
kubectl create namespace logging
## 创建Kubernetes Service account
eksctl create iamserviceaccount \
    --name fluent-bit \
    --namespace logging \
    --cluster <your-cluster-name> \
    --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/fluent-bit-policy" \
    --approve \
    --override-existing-serviceaccounts
## 查看Service Account创建信息
kubectl -n logging describe sa fluent-bit
```
#### 步骤四、获取Flunetbit的IAM role，并配置ElasticSearch访问权限。
```
# 获取Fluentbit role
cat >> ~/.bashrc << EOF
export ES_DOMAIN_USER=<ES_DOMAIN_USER>
export ES_DOMAIN_PASSWORD=<ES_DOMAIN_PASSWORD>
export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster <your-cluster-name> --namespace logging -o json | jq '.iam.serviceAccounts[].status.roleARN' -r)

# 获取Elasticsearch访问端点
export ES_ENDPOINT=$(aws es describe-elasticsearch-domain --domain-name ${ES_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")
EOF

# 更新Elasticsearch访问权限
curl -sS -u "${ES_DOMAIN_USER}:${ES_DOMAIN_PASSWORD}" \
    -X PATCH \
    https://${ES_ENDPOINT}/_opendistro/_security/api/rolesmapping/all_access?pretty \
    -H 'Content-Type: application/json' \
    -d'
[
  {
    "op": "add", "path": "/backend_roles", "value": ["'${FLUENTBIT_ROLE}'"]
  }
]
'
```
#### 步骤五、部署Fluentbit Daemonset。
```
cd ~/environment/logging


# 替换fluentbit文件内的环境变量
curl -Ss https://www.eksworkshop.com/intermediate/230_logging/deploy.files/fluentbit.yaml \
    | envsubst > ~/environment/logging/fluentbit.yaml
# 部署fluentbit
kubectl apply -f ~/environment/logging/fluentbit.yaml
```
#### 步骤六、查看fluentbit部署状态。fluentbit在每台EKS工作节点上运行并收集该工作节点指定目录的日志信息，之后fluentbit会自动将日志自动传输至AWS Elasticsearch服务。
```
kubectl --namespace=logging get pods
```
#### 步骤八、开启EKS管理平面日志。
```
eksctl utils update-cluster-logging --enable-types all --cluster <your-cluster-name> --approve
```




