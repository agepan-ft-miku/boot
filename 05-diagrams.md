# 図の索引と編集用原稿

## 原PNG

[全体ブロック図](diagrams/original/tf_a_v2_12_boot_power_flow.png)と
[処理順序訂正版](diagrams/original/tfa_v2_12_cpu_power_sequence_corrected.png)を原本のまま保存した。
SHA-256、元ファイル名、画素サイズは[画像台帳](diagrams/image-manifest.json)に記録する。

**読む際の補足：CPU0単体OFF/再ONは再始動手段がある場合のみ。**
CPU0に個別リセットがなく代替経路もない条件では、単体OFF対象をCPU1〜3へ限定する。
SYSTEM_OFF/RESETの物理的な実現手段は未確定で、単体OFFとは独立した設計事項。
元画像内の全体矢印や処理一覧は厳密な関数呼び出し順を保証するものではない。

旧図は[履歴](diagrams/history/README.md)へ分離した。実装時は本ページのテキスト図と
[TF-A実装分担](03-tfa-v2.12-port.md)、[電源設計](04-power-reset.md)を使用する。

## Mermaid原稿

以下は会話内容を保存用に整理した編集可能な原稿。`.mmd`と埋め込みは同一内容。
GitHubで表示するためのソースであり、この作業環境ではMermaidエンジンでの描画検証は未実施。

## CPU0のシステム初期起動

[原稿](diagrams/source/01-cold-boot.mmd)

```mermaid
flowchart TD
    A["システム電源投入 / 所定の全体リセット"] --> B["CPU0のみリセット解除"]
    B --> C["Boot ROM等"] --> D["BL2 EL3<br/>DDR・共有HW初期化 / BL31・BL33ロード"]
    D --> E["BL31 cold<br/>EL3常駐環境 / GIC共有・CPU0初期化 / PSCI登録"]
    E --> F["ERET → U-Boot / NS EL2"] --> G["Linux primary / CPU0"]
    G --> H["PSCI CPU_ONでCPU1〜3を順次起動"]
    R["CPU1〜3は初期ブート中リセット保持"] -.-> H
    N["前提: AArch64 / BL32なし / 1クラスタ4CPU"] -.-> D
```

## PSCI CPU_ON：要求元と起動先

[原稿](diagrams/source/02-cpu-on.mmd)

```mermaid
sequenceDiagram
    participant L as Linux / 稼働CPU m
    participant P as BL31 / 要求元CPU m
    participant H as SoC reset control
    participant T as BL31 / 対象CPU n
    participant K as Linux / 対象CPU n
    L->>P: SMC CPU_ON(MPIDR n, NS entry PA, context ID)
    P->>P: 検証・CPUロック・ON_PENDING
    P->>H: pwr_domain_on(n): HW状態確認・ベクタ設定
    Note over P,H: reset vector = sec_entrypoint (BL31 warm)
    P->>H: 書き込み完了保証・対象CPUのみリセット解除
    H->>T: bl31_warm_entrypoint
    par 要求元
        P->>P: pwr_domain_on成功後にNSコンテキストを初期化
        P->>P: CPUロック解放
        P-->>L: CPU_ONの結果
    and 起動先
        T->>T: A53・EL3・スタック・MMU初期化
        T->>T: pwr_domain_on_finish()
        T->>T: 本案では汎用処理でD-cache有効化
        T->>T: pwr_domain_on_finish_late(): GICR / CPU IF
        T->>T: アーキテクチャ準備・要求元ロックと同期
        T->>K: ERET → secondary_entry
        K->>K: CPU別初期化 → Online
    end
    Note over L,K: CPU_ONの復帰とLinux Onlineは別の完了点
    Note over P,T: CPU0再ONはHW・OSが対応する場合のみ
```

## PSCI CPU_OFF：停止対象CPU自身の処理

[原稿](diagrams/source/03-cpu-off.mmd)

```mermaid
flowchart TD
    A["対象CPUのLinux<br/>ホットプラグ停止準備"] --> B["対象CPU自身がSMC CPU_OFF"]
    B --> C["psci_do_cpu_off<br/>pwr_domain_off_early: 任意の早期拒否"]
    C --> D{"停止を許可するか"}
    D -->|拒否| X["DENIED<br/>CPU0禁止はLinux側でも早期に扱う"]
    D -->|許可| E["親ドメイン・ロック・状態協調"]
    E --> F["psci_pwrdown_cpu<br/>アーキテクチャ・キャッシュ停止準備"]
    F --> G["pwr_domain_off<br/>GIC CPU IF停止 / HW停止予約・準備"]
    G --> H["汎用処理<br/>ロック解放 / affinity OFF反映・同期"]
    H --> I["pwr_domain_pwr_down_wfi<br/>またはpsci_power_down_wfi"]
    I --> J["SoC仕様に基づく安全なHW停止"]
    J --> K["別CPUのCPU_ONでwarm再起動<br/>再始動手段があるCPUのみ"]
    N["pwr_domain_offの途中で即時自己リセットしない<br/>WFIやリセット保持は物理電源断ではない"] -.-> G
```

## システム全体のOFFとRESET

[原稿](diagrams/source/04-system-power.mmd)

```mermaid
flowchart TD
    A["Linuxの終了・再起動処理<br/>最後の稼働CPUで実行"] --> B{"操作"}
    B -->|SYSTEM_OFF| C["BL31 psci_system_off → system_off"]
    C --> D{"電源断機構あり"}
    D -->|あり| E["PMIC / 常時給電回路へ要求"] --> F["SoC物理電源OFF"]
    D -->|なし| G["停止・通電継続のみ<br/>物理電源OFFは未対応"]
    B -->|SYSTEM_RESET| H["BL31 psci_system_reset → system_reset"]
    H --> I{"CPU0を含む全体リセットあり"}
    I -->|あり| J["全体リセット / 対応Watchdog / 外部回路"]
    J --> K["CPU0 cold起動<br/>Boot ROM等 → BL2 → BL31 → U-Boot → Linux"]
    I -->|なし| L{"外部回路の自動電源サイクルあり"}
    L -->|あり| M["OFF→自動ON"] --> K
    L -->|なし| O["通常のHW再起動は未対応<br/>外部操作等が必要"]
    N["CPU0個別リセット・CPU0単体OFFとは別機能<br/>通常の終了経路にU-Bootは入らない"] -.-> B
```

## CPU0に個別リセットがない場合の判断

[原稿](diagrams/source/05-cpu0-constraint.mmd)

```mermaid
flowchart TD
    A["評価条件: CPU0の個別リセット機能なし"] --> B{"別の単体再始動手段あり"}
    B -->|あり| C["CPU0 hotplugを別途設計・検証<br/>CPU_ON時はwarm経路"]
    B -->|なし| D["CPU0を単体OFF対象から除外"]
    D --> E["Linuxで早期拒否<br/>TF-A off_earlyでも保護"]
    A --> F{"CPU0を含む全体リセットあり"}
    F -->|あり| G["SYSTEM_RESETは実装可能"]
    F -->|なし| H["自動電源サイクル等を確認<br/>なければ通常再起動は外部操作"]
    A --> I{"SoC / ボードの電源断機構あり"}
    I -->|あり| J["SYSTEM_OFFは実装可能"]
    I -->|なし| K["停止・通電継続まで"]
    N["各機構の実際の有無は未確定<br/>個別リセットなしから全体リセットなしを推定しない"] -.-> A
```

## GIC-500の共有初期化とCPU別初期化

[原稿](diagrams/source/06-gic-split.mmd)

```mermaid
flowchart LR
    subgraph C["CPU0 / BL31 cold"]
        A["gicv3_driver_init"] --> B["gicv3_distif_init"]
        B --> D["gicv3_rdistif_init CPU0"] --> E["gicv3_cpuif_enable CPU0"]
    end
    subgraph W["対象CPU / BL31 warm"]
        F["pwr_domain_on_finish"] --> G["汎用D-cache有効化: 本案"]
        G --> H["pwr_domain_on_finish_late"]
        H --> I["gicv3_rdistif_init 対象CPU"] --> J["gicv3_cpuif_enable 対象CPU"]
    end
    N["GICR対応は事前確定<br/>CPU別探索方式なら初期化前にprobe"] -.-> I
    M["セカンダリでDistributor共有初期化を繰り返さない"] -.-> W
```
