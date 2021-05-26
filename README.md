# ポートフォリオ
　CI、CDツールであるJenkinsを用いて自動インフラ構築、自動デプロイ、自動テストを実践する

# 使い方
### 1. Jenkins実行用サーバーを構築
　CFnのテンプレートをGitクローンする
```
git clone　https://github.com/kosukeoku/cloudformation.git
```
　AWS CFnでスタックの作成をクリック. 
　先ほどクローンしたjenkins.ymlをアップロードし、スタック名、PJPrefixを入力しスタックの作成を行う

　作成したEC2インスタンスにアクセスする
```
ssh -i {ペアキー} ec2-user@{パブリックIPアドレス}
```
```
sudo yum install git
```
　Jenkinsの初期パスワードをコピーする
```
sudo less /var/lib/jenkins/secrets/initialAdminPassword
```
　Jenkinsの初期設定を行う. 
　Jenkinsの実行ユーザーを変更する. 
  /etc/sysconfig/jenkins に. 
```
JENKINS_USER="ec2-user"
```
　所有権を変更
```
sudo chown -R ec2-user: /var/lib/jenkins /var/log/jenkins /var/cache/jenkins
```

### 2. CFnのジョブを作成する
　AWS CLIを使用するためにCLIの設定を行う
  ```
  $ aws configure --profile user1
AWS Access Key ID [None]: {アクセスキー(各自)}
AWS Secret Access Key [None]: {シークレットアクセスキー(各自)}
Default region name [None]: ap-northeast-1
Default output format [None]: json
  ```
　アプリ名の環境変数の追加.   
　　①Jenkins のホーム ページで [Jenkinsの管理]  
　　②[システムの設定]   
　　③[グローバル プロパティ] セクションで、[環境変数] をオンにします。    
　　④[追加] をクリックし、キーにAPP_NAME、値にアプリの名前を入力.   

　GitとRDS認証情報の追加.   
　　①Jenkins のホーム ページで [Jenkinsの管理]  
　　②Credential→domain(global).   
　　③認証情報の追加  
　　④ユーザー名とパスワードで認証情報を追加する. 　　

　新規ジョブを作成.   
　execute_cloudformationを入力しフリースタイルプロジェクトのビルドを選択しOK. 
　ソースコード管理でGitを選択し以下を入力する. 
 ```
 https://github.com/kosukeoku/cloudformation.git
 ```
　ビルドするブランチは
 ```
 */main
 ```
 
　ビルド環境⇨秘密ファイルや秘密テキストを使用する
 ```
 #ユーザー名変数
 RDS_USER
 #パスワード変数
 RDS_PASS
 #RDSの認証情報を選択
 ```
　ビルド手順の追加、シェルの実行を選択しシェルスクリプトに以下を記入
 ```
sudo -u ec2-user aws cloudformation create-stack \
--stack-name $APPNAME \
--template-body file://$WORKSPACE/app-template.yml \
--parameters ParameterKey='PJPrefix',ParameterValue=$APPNAME \
ParameterKey='DBMasterUserName',ParameterValue=$RDS_USER \
ParameterKey='DBPassword',ParameterValue=$RDS_PASS

sudo -u ec2-user aws cloudformation wait stack-create-complete --stack-name $APPNAME
 ```
　
 ### 2. Ansibleのジョブを作成する
 
 pemファイルをjenkinsサーバーにアップロードする.   
 ```
 scp -i ~/.ssh/<pemファイル> <アップロードするpemファイル> ec2-user@<デプロイするEC2のパブリックIP>:/home/ec2-user/.ssh/
 ```
 
 GitHubでSSH接続するために公開鍵・秘密鍵を生成する
 ```
 cd ~/.ssh/
 ssh-keygen -t rsa
 ```
 execute_ansibleを入力しフリースタイルプロジェクトのビルドを選択しOK.   
 ジョブを作成後にワークスペースに移動.   
 ```
 cd /var/lib/jenkins/workspace/execute_ansible
 ```
 ワークスペースでリポジトリをクローン
 ```
 git clone https://github.com/kosukeoku/ansible.git
 ```
 ansibleの設定は https://github.com/kosukeoku/ansible.git のREADMEに記載済み. 
 ※private.ymlにはdb,ec2-ipはjenkinsから自動入力するので書かなくて良い. 
 作成した変数ファイルを暗号化する
 ```
 ansible-vault encrypt private.yml
 ```
 Jenkinsでexecute_ansibleの設定をクリックする。    
 ビルド環境⇨秘密ファイルや秘密テキストを使用する
 ```
 #ユーザー名変数
 RDS_USER
 #パスワード変数
 RDS_PASS
 #RDSの認証情報を選択
 ```
ビルドのシェルスクリプトには以下を入力する
``` 
ansible-vault decrypt private.yml
EC2_IP=`aws ec2 describe-instances --filter "Name=tag:Name,Values=$APPNAME-ec2-instance" "Name=instance-state-name,Values=running" --query "Reservations[].Instances[].[PublicIpAddress]" --output text`
sed -i "s/\(HostName\)\(.*\)/\1\ $EC2_IP/" ~/.ssh/config
sed -i "s/\(ec2_ip:\)\(.*\)/\1\ $EC2_IP/" /var/lib/jenkins/workspace/execute_ansible/private.yml
RDS_ENDPOINT=`aws rds describe-db-instances --db-instance-identifier $APPNAME-rds-instance --query "DBInstances[].[Endpoint.Address]" --output text`
sed -i "s/\(db_host:\)\(.*\)/\1\ $RDS_ENDPOINT/" /var/lib/jenkins/workspace/execute_ansible/private.yml
sed -i "s/\(db_username:\)\(.*\)/\1\ $RDS_USER/" /var/lib/jenkins/workspace/execute_ansible/private.yml
sed -i "s/\(db_password:\)\(.*\)/\1\ $RDS_PASS/" /var/lib/jenkins/workspace/execute_ansible/private.yml
ansible-vault encrypt private.yml
ansible-playbook -i inventory site.yml
```
　 ### 2. Serverspecのジョブを作成する
