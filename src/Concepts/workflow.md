AnonifyネットワークにTEEが参加する登録フェーズ、外部からAnonifyへトランザクションを送信する状態遷移フェーズ、そして暗号化に用いられるグループ鍵をローテーションする鍵ローテーションフェーズのワークフローを紹介します。ここでは、外部からトランザクションを送信する主体は、Enclaveへアクセス可能な鍵などの認証情報を保持していることを前提とします。

## 登録フェーズ
TEEが正規の状態遷移プログラムを実行することを検証した上でブロックチェーンに公開鍵を登録するフェーズです。Remote Attestationにより得られたREPORTのオンチェーン検証とそれに含まれているEnclave Identity公開鍵のオンチェーンへの登録を行います。REPORTの検証には、REPORTの署名検証やMRENCLAVEの一致検証などが含ます。登録フェーズは、特定の状態遷移ロジックを決め、新しいAnonifyネットワークを生成するとき、TEEが既存のAnonifyネットワークに参加するとき、REPORTタイムスタンプの有効期限が切れたときに実施されるフェーズです。
TEEがEnclaveを生成してから、Anonify上の特定のグループに参加するまでのワークフローは以下のようになります。

* Enclave生成時にEnclaveに閉じたIdentity鍵ペアを生成
* Remote Attestationを行う.QUOTEにIdentity公開鍵やNonceを含め,Attestation Serviceに送信し署名付きREPORTのレスポンスを受け取る
* グループ鍵を共有するためのハンドシェイクを生成する
* REPORTとハンドシェイクを含むトランザクションを生成しブロックチェーンにブロードキャスト
* ブロックチェーン上でのREPORTの一連の検証を通して,Identity公開鍵をブロックチェーンに記録することでTEEの登録が完了する.REPORTの検証において,新しいAnonifyネットワークを生成する場合は,ブロックチェーンにMRENCLAVEを記録する.一方,既存のAnonifyネットワークに参加する場合は,すでにブロックチェーンに記録されているMRENCLAVEと提示しているMRENCLAVEが一致するか検証を行う.

## 状態遷移フェーズ
暗号化された状態遷移の結果を共有するためにトランザクションをブロックチェーンネットワークでブロードキャストするフェーズです。
認証情報を持つ外部のエンティティがAnonify上でトランザクションを送信し、ブロックチェーンで共有され他のTEEの状態に反映されるワークフローは以下のようになります。

* 外部エンティティがTEEに対してアクセスに必要な認証情報と状態遷移のインプットを送信する
* Enclave内で認証情報の検証を行い,決められた状態遷移関数にデータをインプットし計算を行う
* TEEはその結果を暗号化,ロックパラメタの計算,Enclave Identity秘密鍵による署名をし,ブロックチェーンノードにこれらのデータを含めたトランザクションをブロードキャストする
* ブロックチェーン上のスマートコントラクトでEnclave Identity公開鍵による署名検証後,暗号文を記録する
* ネットワーク内の全てのTEEはブロックチェーンに記録された暗号文を取得しEnclave内で復号し状態を更新する


## 鍵ローテーションフェーズ
全てのTEEで共有しているグループ暗号鍵をローテーションするフェーズです。ワークフローは以下のようになります。

* TEEの運営者クライアントが鍵ローテーションのためのリクエストをTEEへ送信する
* リクエストに合わせて新しい鍵を内部で生成あるいは外部から取得を行う
* 新しい鍵を用いてTreeKEMのグループ状態アップデートを計算する
* TreeKEMの新しい状態を他のTEEに伝えるためにメンバTEEの公開鍵を用いて暗号化しブロックチェーンへブロードキャストする
* それぞれのTEEはフェッチした暗号文を自身の秘密鍵で復号、TreeKEMのグループ状態をアップデートし,それぞれ共通のグループシード値が得られる
