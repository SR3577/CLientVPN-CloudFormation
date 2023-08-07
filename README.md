# ClientVPNをCloudFormationを使って構築する

## 概要

- AWSのClientVPNをCloudFormationで作成したもの。

- 証明書の作成やダウンロードなど、完全にコード化はできなかったが手作業よりミスが減るはず。

- AWSのClientVPNを使用して社内で使用するサーバーの構築をしてほしいと依頼されたため調査、作成。

- 調査していると、AWSの「AWS Hands-on for Beginners」にちょうど良いものがあったため、同様のものを検証用に構築。

- 構築後は構成図のEC2を要望どおりの仕様にした。

## 構成図

![01_Diagrams](https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/e36641af-9a7b-4d5d-b70b-7aeb46fa27ed)


## 手順

### VPC～サブネット作成

01_network.yml

### EC2の作成

02_ec2.yml

### ロググループ～ログストリームの作成

03_cloudwatch.yml

##### 参考

[AWS::Logs::LogGroup - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-logs-loggroup.html)

[AWS::Logs::LogStream - AWS CloudFormation](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-logs-logstream.html)

### 証明書の作成

マネジメントコンソールのCloudShellで実行

```bash
sudo yum -y install openssl
```

#### 以下は下記ドキュメントを参照

[相互認証 - AWS クライアント VPN](https://docs.aws.amazon.com/ja_jp/vpn/latest/clientvpn-admin/mutual.html)

```bash
git clone https://github.com/OpenVPN/easy-rsa.git
```

```bash
cd easy-rsa/easyrsa3
```

```bash
./easyrsa init-pki
```

```bash
./easyrsa build-ca nopass
```

```bash
./easyrsa build-server-full server nopass

#入力を求められたらEnter
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:
```

```bash
./easyrsa build-server-full server nopass

#入力を求められるため、yesと入力
Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
```

```bash
./easyrsa build-client-full client1.domain.tld nopass

#入力を求められるため、yesと入力
Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
```

```bash
mkdir ~/custom_folder/
cp pki/ca.crt ~/custom_folder/
cp pki/issued/server.crt ~/custom_folder/
cp pki/private/server.key ~/custom_folder/
cp pki/issued/client1.domain.tld.crt ~/custom_folder
cp pki/private/client1.domain.tld.key ~/custom_folder/
cd ~/custom_folder/
```

```bash
#確認
ls -l

-rw------- 1 cloudshell-user cloudshell-user 1172 Aug ca.crt
-rw------- 1 cloudshell-user cloudshell-user 4460 Aug client1.domain.tld.crt
-rw------- 1 cloudshell-user cloudshell-user 1704 Aug client1.domain.tld.key
-rw------- 1 cloudshell-user cloudshell-user 4552 Aug server.crt
-rw------- 1 cloudshell-user cloudshell-user 1704 Aug server.key
```

サーバー証明書とキー、およびクライアント証明書とキーを ACM にアップロード

```bash
aws acm import-certificate --certificate fileb://server.crt --private-key fileb://server.key --certificate-chain fileb://ca.crt
```

```bash
aws acm import-certificate --certificate fileb://client1.domain.tld.crt --private-key fileb://client1.domain.tld.key --certificate-chain fileb://ca.crt
```

#### 証明書をダウンロード

- 以下２つをダウンロードする

  client1.domain.tld.crt
  
  client1.domain.tld.key

- アクション⇒ファイルのダウンロードを選択
    
- 下記のとおり入力
    
    ```bash
    ~/custom_folder/client1.domain.tld.crt
    ```
    
- 別の証明書をダウンロード
    
    ```bash
    ~/custom_folder/client1.domain.tld.key
    ```
    
### ClientVPNエンドポイント作成

04_clientvpn.yml

#### クライアント設定をダウンロード

<img width="370" alt="02_downloadClientSetting" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/7c97e4d2-bdc7-4cb7-8b60-b276a331d8e7">


<img width="374" alt="03_downloadClientSettingStart" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/c2b79b73-2ce5-40a0-b05b-ecb239985094">

### 接続するための準備

現在手元には下記３ファイルがある

 - client1.domain.tld.crt
  
 - client1.domain.tld.key

 - downloaded-client-config.ovpn

#### ダウンロードしたファイルを編集する

- downloaded-client-config.ovpnの最下部に`<cert></cert>`と入力
    
- client1.domain.tld.crtの----BEGIN CERTIFICATE---—から----END CERTIFICATE---—（BIGINとかも含む）をコピー
        
- downloaded-client-config.ovpnの<cert></cert>の間にペースト
        
- downloaded-client-config.ovpnの最下部に`<key></key>`と追加
    
- client1.domain.tld.keyの中身全てをコピー
    
- downloaded-client-config.ovpnの`<key></key>`の間にペースト（改行などが入らないようにする）
    
### 接続する

- AWS VPN Clientを開く
    
- ファイル⇒プロファイルを管理をクリック

  <img width="347" alt="04_addProfile" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/0963d0fc-a6f5-4664-aff9-371dd53752a8">

- プロファイルを追加

  <img width="279" alt="05_addProfile2" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/435ae868-cb30-40f4-b78a-f1240ee6f9af">

- downloaded-client-config.ovpnを選択して「プロファイルを追加」

  <img width="354" alt="06_selectOvpnFile" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/c32b5def-e76e-4b60-a764-f4d41b8dcdb1">


- 作成したプロファイルを選択して、「接続」

  <img width="264" alt="07_startConnection" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/a6cd87b8-a2fa-4306-8498-e3d5fa3376f7">

- 接続できている

  <img width="353" alt="08_connectSuccess" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/860d044a-862b-4ef0-9c31-945466d843ca">


### 接続確認

- プライベートIPアドレスでping

  <img width="435" alt="09_pingSuccess" src="https://github.com/SR3577/CLientVPN-CloudFormation/assets/138550117/acfd46fc-71be-4eaa-b150-a9fff7f248cb">


- VPNエンドポイントから、「接続」タブを選択すると接続状況がわかる
    
- CloudWatchにもLogが残っていることが確認できる（反映に少し時間かかる）
