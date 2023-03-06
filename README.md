# 内容

awsのクライアントVPNエンドポイントから固定IPでインターネット接続をするメモ


# 相互認証のための証明書作成

https://docs.aws.amazon.com/ja_jp/vpn/latest/clientvpn-admin/client-authentication.html#mutual

```
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
./easyrsa init-pki
./easyrsa build-ca nopass
> Enter
./easyrsa build-server-full sample-vpn-server nopass
> yes
./easyrsa build-client-full sample-vpn-client1.domain.tld nopass
> yes

# 別フォルダに必要なものをコピー
mkdir ~/sample-vpn/
cp pki/ca.crt ~/sample-vpn/
cp pki/issued/sample-vpn-server.crt ~/sample-vpn/
cp pki/private/sample-vpn-server.key ~/sample-vpn/
cp pki/issued/sample-vpn-client1.domain.tld.crt ~/sample-vpn
cp pki/private/sample-vpn-client1.domain.tld.key ~/sample-vpn/
cd ~/sample-vpn

# ACMに登録
aws acm import-certificate --certificate fileb://sample-vpn-server.crt --private-key fileb://sample-vpn-server.key --certificate-chain fileb://ca.crt
aws acm import-certificate --certificate fileb://sample-vpn-client1.domain.tld.crt --private-key fileb://sample-vpn-client1.domain.tld.key --certificate-chain fileb://ca.crt
```

# AWS構成

- VPC作成
- サブネット作成
  - private-subnet
  - public-subnet
- nat-gateway作成
  - public-subnet内
  - Elastic IPあり
- internet-gateway作成
- ルートテーブル
  - private-subnet -> 0.0.0.0/0 nat-gateway
  - public-subnet -> 0.0.0.0/0 internet-gateway
- クライアントVPNエンドポイント作成
  - サーバー証明書 ACMのserver
  - 相互認証 -> クライアント証明書 ACMのclient
  - スプリットトンネル無効
  - ターゲットネットワークをprivate-subnet
  - 承認ルール 0.0.0.0/0追加
  - ルートテーブル 0.0.0.0/0(private-subnet)追加

# ovpnファイルの作成

作成したクライアントVPNエンドポイントからクライアント設定`**.ovpn`をダウンロード
編集して`sample-vpn-client1-domain.tld.crt`, `ample-vpn-client1-domain.tld.key`の情報を追記

```
<ca>
-----BEGIN CERTIFICATE-----
***
-----END CERTIFICATE-----

</ca>

<cert>
-----BEGIN CERTIFICATE-----
***
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
***
-----END PRIVATE KEY-----
</key>
```

# OpenVPNクライアント

https://docs.aws.amazon.com/ja_jp/vpn/latest/clientvpn-user/windows.html

- OpenVPN GUP for windows
  - https://openvpn.net/community-downloads/
- Tunnelblick for macOS
  - https://docs.aws.amazon.com/ja_jp/vpn/latest/clientvpn-user/macos.html