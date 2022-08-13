# sbcntr-resources

書籍用の各種リソースのダウンロードリポジトリです。

## リポジトリの構成

```bash
❯ tree . -d 1
.
├── cicd
├── cloudformations
├── cloud9
├── fargate-bastion
├── firelens
├── iam
└── scan
```

それぞれのディレクトリと本書の関連付けは次の通りです。

| ディレクトリ          | 本書の対象箇所                                                    | 内容                                               |
|-----------------|------------------------------------------------------------|--------------------------------------------------|
| cicd            | 5-2 運用設計:Codeシリーズを使った CI/CD                                | CI/CD設計で利用するビルド定義やタスク定義などを格納                     |
| cloudformations | 4-2 ネットワークの構築, 4-4 コンテナレジストリの構築, 4-5 オーケストレータの構築, 付録1 | ハンズオンや付録で利用するCloudFormationを格納                      |
| cloud9          | 4-4 コンテナレジストリの構築   | resize.shを格納 |
| fargate-bastion | 5-7 運用設計:FargateによるBastion(踏み台ホスト)の構築                      | Fargate Bastionで利用するBastionコンテナの設定を格納            |
| firelens        | 5-6 運用設計 &セキュリティ設計:ログ収集基盤の構築                               | ログ収集基盤のFireLensコンテナに設定するログ設定を格納                  |
| iam             | 4章、5章全般                                                    | 4章や5章の各所で利用するIAMポリシーのJSONファイルを格納                 |
| scan            | 5-8 セキュリティ設計:  Trivy/Dockleによるセキュリティチェック                   | DevSecOpsを実現するためのCodeBuildのビルド定義を格納              |

本書でPDFからコードをコピー＆ペーストしたり、写経するのが難しい場合がありますので、是非ご活用ください。

# 20220813追記

# docker操作

## dockerイメージ確認
```
docker image ls
```

## dockerイメージ削除
```
docker image rm -f $(docker image ls -q)
```

# backendアプリビルド

## dockerビルド
```
cd /home/ec2-user/environment/sbcntr-backend
docker image build -t sbcntr-backend:v1 .
```

## awsアカウントID取得
```
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
```

## イメージタグ修正
```
docker image tag sbcntr-backend:v1 ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-backend:v1
```

## Docker認証
```
aws ecr --region ap-northeast-1 get-login-password | docker login --username AWS --password-stdin https://${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-backend
```

## ECRへのイメージ登録
```
docker image push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-backend:v1
```

# backend動確

## dockerイメージ削除
```
docker image rm -f $(docker image ls -q)
```

## dockerイメージ取得
```
docker image pull ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-backend:v1
```

## docker コンテナ起動
```
docker container run -d -p 8080:80 ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-backend:v1
```

## リクエスト確認
```
date; curl http://localhost:8080/v1/helloworld

結果:
Sat Aug 13 03:45:13 UTC 2022
{"data":"Hello world"}
```

## コンテナ停止
```
docker stop $(docker ps -q)
docker ps
```

# frontendアプリビルド

## dockerビルド
```
cd /home/ec2-user/environment/sbcntr-frontend
docker image build -t sbcntr-frontend:v1 .
docker image build -t sbcntr-frontend:dbv1 .
```

## awsアカウントID取得
```
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
```

## イメージタグ修正
```
docker image tag sbcntr-frontend:v1 ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-frontend:v1
docker image tag sbcntr-frontend:dbv1 ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-frontend:dbv1
```

## Docker認証
```
aws ecr --region ap-northeast-1 get-login-password | docker login --username AWS --password-stdin https://${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-frontend
```

## ECRへのイメージ登録
```
docker image push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-frontend:v1
docker image push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/sbcntr-frontend:dbv1

```

# 稼働確認

## バックエンド確認
```
curl http://internal-sbcntr-alb-internal-1598506546.ap-northeast-1.elb.amazonaws.com/v1/helloworld
{"data":"Hello world"}
curl http://internal-sbcntr-alb-internal-1598506546.ap-northeast-1.elb.amazonaws.com/v1/Notifications?id=1
{"data":[{"id":1,"title":"通知1","description":"コンテナアプリケーションの作成の時間です。","category":"information","unread":true,"createdAt":"2022-08-13T05:57:14.925+09:00","updatedAt":"2022-08-13T05:57:14.925+09:00"}]}
```

## フロントエンド確認
```
http://awsnaritomo.com/
```

# DB設定

## ユーザ/ロール作成
```
mysql -h sbcntr-db.cluster-cr5ewonryu9t.ap-northeast-1.rds.amazonaws.com -u admin -p5KBSJn8sUvPebhcBjLGi
SELECT Host, User FROM mysql.user;
CREATE USER sbcntruser@'%' IDENTIFIED BY 'sbcntrEncP';
GRANT ALL ON sbcntrapp.* TO sbcntruser@'%' WITH GRANT OPTION;
CREATE USER migrate@'%' IDENTIFIED BY 'sbcntrMigrate';
GRANT ALL ON sbcntrapp.* TO migrate@'%' WITH GRANT OPTION;
GRANT ALL ON `prisma_migrate_shadow_db%`.* TO migrate@'%' WITH GRANT OPTION;
SELECT Host, User FROM mysql.user;
```

## テーブル確認
```
mysql -h sbcntr-db.cluster-cr5ewonryu9t.ap-northeast-1.rds.amazonaws.com -u sbcntruser -psbcntrEncP
exit
mysql -h sbcntr-db.cluster-cr5ewonryu9t.ap-northeast-1.rds.amazonaws.com -u migrate -psbcntrMigrate
use sbcntrapp;
show tables;
exit;
```

## テーブル作成/データ投入
```
cd /home/ec2-user/environment/sbcntr-frontend
git checkout main
export DB_USERNAME=migrate
export DB_PASSWORD=sbcntrMigrate
export DB_HOST=sbcntr-db.cluster-cr5ewonryu9t.ap-northeast-1.rds.amazonaws.com
export DB_NAME=sbcntrapp
npm run migrate:dev

? Name of migration → init

npm run seed
```

## テーブル作成/データ投入確認
```
mysql -h sbcntr-db.cluster-cr5ewonryu9t.ap-northeast-1.rds.amazonaws.com -u sbcntruser -psbcntrEncP
use sbcntrapp;
show tables;
select * from Notification;
exit;
```
