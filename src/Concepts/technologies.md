Anonifyを構成する主要な技術的要素について紹介します。

## ブロックチェーン

ブロックチェーン技術はネットワークに参加するノードが状態遷移の正しさを互いに検証し、暗号学的に改ざん不可能な状態データとして記録、共有することを可能にします。Anonifyでは、TEEで実行される状態遷移の完全性を検証する処理を行う基盤として、そして暗号化された状態遷移の履歴を複数主体間で共有するためにブロックチェーンを用いています。
暗号化された状態遷移の履歴はノード間でレプリケーションされ、復号鍵を保持しているTEEはいつでも合意された最新の状態にアクセスすることが可能です。

## TEE (Trusted Execution Environment)

TEE (Trusted Execution Environment)は特定のアプリケーションが他のソフトウェアから隔離されたEnclaveで実行されることを保証するプロセッサの拡張機能である。TEEの基本的なセキュリティ要件とインターフェース定義は標準化団体[GlobalPlatform](https://globalplatform.org/)が行なっています。

TEEではハードウェアレベルの設計により、カーネルOS、ハイパーバイザなどの権限の強いシステムソフトウェアが保護領域のメモリに対して不正にアクセスできないようになっています。つまり、システムソフトウェアからのマルウェアに対してもアプリケーションが保護されます。このようなセキュリティ性質により、従来より格段に隔離性が高い環境で汎用プログラムを安全に実行できるため、機密性の高いデータを伴うアプリケーションへの応用が進められています。

代表的なTEEとして、Intel SGX、ARM TrustZone、RISC-V Keystoneなどがあげられ、AWS Nitro Enclavesも同等の性質を持ちます。また、メモリの隔離保護に加えAttestationと呼ばれる機能を持つTEEもあり、この機能により意図した実行バイナリが正規プロセッサで動作していることを保証します。まとめると、TEEは実行バイナリの秘匿性と完全性を提供する性質を持つことになります。

また、TPM (Trusted Platform Module) もプロセッサレベルの隔離実行機能の代表的な標準規格です。TPMはプログラムがチップセットにハードコードされているのに対し、TEEは一般開発者に隔離実行するプログラムの実装が拓かれている点で大きく異なります。また、Google Titan、 Apple T2、 Microsoft Plutonも同様の機能を持ちますが、実装とアップデートはあくまでプロダクトベンダのみに制限されてしまっています。

### Intel SGX

Intel SGX (Software Guard Execution)は第六世代のIntel Skylakeマイクロアーキテクチャ以降に搭載されているプロセッサにおけるセキュリティ機能です。プロセッサ起動時にSGXにおいては固定サイズの保護領域がDRAMのサブセットとして確保され、Enclave領域として仮想メモリが割り当てられます。保護領域はプロセッサによりアクセスコントロールされ、システムソフトウェアやDMA経由ですらアクセスすることはできません。

また、サイドチャネル攻撃耐性のためにEnclave内のメモリは暗号化した上で記録され、プロセッサでの処理時に再度復号されます。さらに、Enclave内のソフトウェアはリングプロテクションにおけるRing3 (ユーザランド)の権限でしか命令を実行することはできず、Ring0のような強い権限でアプリケーションを実行することができないようになっています。Enclave内外へのアクセスをする特有の命令であるEENTERやEEXITもRing3権限の命令です。

SGXの特徴的な機能としてSealingとAttestationがあげられます。Sealingはon-chipの鍵とEnclaveでのプログラムに依存する鍵を導出し、その鍵を用いてEnclaveで暗号化することができます。そして、その暗号文は外部にエクスポートしディスクに保存することが可能です。復元したい場合は同様に暗号文をEnclave内にインポートし、同じ鍵でUnsealingを行い復号することができます。もう一つの機能的性質であるAttestationについては以下で紹介します。

### Attestation

Intel SGXの秘匿性はEnclaveの隔離保護により提供されますが、これだけではプロセッサやEnclave内で動作するプログラムの完全性は第三者が検証することができません。
Remote Attestationの機能はこの完全性の性質を提供します。つまり、特定のEnclave内で確かに正しい実行バイナリが動作していることを遠隔のAttestation Serviceが保証します。
対して、Local Attestationは同一マシン上のEnclave同士が確かに同一マシン上で動作していることを互いに検証するためのプロトコル機能ですが、特にここではAnonifyで重要なRemote Attesttaionの機能を紹介します。

Remote Attestationは、マシン外部のAttestation Serviceに特定のSGXプロセッサのセキュリティステータスやEnclave実行バイナリを含むデータ構造体QUOTEを送信し、Attestation Serviceが保持する秘密鍵により署名された結果データREPORTを得ることができます。Attestation ServiceはIntelが運営するIAS (Intel Attestation Service) やサードパーティが運営するDCAP (Data Center Attestation Primitives) を利用することが可能です。
このREPORTにはEnclave内実行バイナリや実行環境に依存したハッシュ値 (MRENCLAVE) が含まれ、同じソースコードで同じ設定でビルドすれば同じMRENCLAVEを得ることができます。

## TreeKEM

TreeKEMは、複数のメンバ間でグループ鍵をバイナリツリー構造に基づき効率的に生成・共有するためのKEM (Key Encapsulation Mechanism) です。KEMは、HPKE (Hybrid Public Key Encryption) などの暗号プリミティブを用いて共通鍵を暗号化する仕組みのことを言います。TreeKEMはMLS (Messaging Layer Security) として知られるグループメッセージングのEnds-to-Ends Encryption (E2EE) プロトコルのコアとなる技術であり、IETF Working Groupのドラフトとして標準化が進められています。

メンバ間の一連の暗号化メッセージのやり取りがあるフローにおいて、ある特定の暗号化メッセージの鍵が漏洩しても、Forward Secrecyによりそのメッセージ以前の暗号文が解読不可能なことを保証し、Post-Compromise Securityによりそれ以降の暗号文が解読不可能なことを保証します。つまり、グループ間でのコミュニケーションにおいて、特定の暗号文に対する鍵が漏洩しても他の暗号文にセキュリティ的影響を与えないことを保証します。このMLSで中心となる技術的要素がツリー構造のグループ鍵決定メカニズムであるTreeKEMとなっています。
AnonifyではTreeKEMを用いることで、全てのTEEで使う共通のグループ鍵を外部に漏洩することなく効率的に共有することを可能にします。これにより、グループへのメンバの加入、脱退、鍵更新を従来の手法と比べ効率的に実現します。そして、このグループ鍵に基づき暗号化した状態遷移の結果をブロックチェーンに記録します。
