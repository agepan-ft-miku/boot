# CPUのON/OFFとシステムOFF/RESET

**CPU単体のPSCI状態、CPUリセット状態、クロック状態、物理電源状態を区別する。**
個別リセットレジスタの存在だけでは物理電源OFFを実装できるとは限らない。
本資料のCPU単体操作は、DDR/BL31/共有資源を保持し、他CPUが動いている条件を対象とする。

## 1. CPU_ON：要求元CPUが対象CPUを起動

要求元はCPU0固定ではなく、稼働CPU。
CPU0のcoldと異なり、対象CPUはBL31 warmから起動する。

```text
Linux CPU_ON(target, Linux entry PA, context ID)
  → 汎用PSCI：検証・CPUロック取得・ON_PENDING設定
  → pwr_domain_on(target)
      1. CPU番号とHW状態を確認
      2. 前回OFFのHW完了を確認（再ONの場合）
      3. 必要な電源・クロックを準備し、対象CPUはリセット保持
      4. reset vector = sec_entrypoint
      5. 書き込み完了/read-back/必要なバリアを保証
      6. 対象CPUのみリセット解除
      7. 成功を返す
  → 汎用PSCI：NSコンテキスト初期化・ロック解除・復帰
```

対象CPU側：

```text
bl31_warm_entrypoint → A53/EL3/スタック/MMUの初期化
  → psci_warmboot_entrypoint → psci_cpu_on_finish
  → pwr_domain_on_finish
  → 汎用D-cache有効化（本案の設定）
  → pwr_domain_on_finish_late：CPU別GIC
  → アーキテクチャ準備・要求元との同期・NSへERET
  → Linux secondary_entry → Online
```

`pwr_domain_on()`内でLinux Onlineを待たない。
戻り値が成功した後にNSコンテキストが設定されるv2.12の同期経路を壊さない。
開始済みCPUに対し失敗を返すことも避ける。[S-ON](08-references.md)

## 2. CPU_OFF：対象CPU自身が停止

Linuxのホットプラグ制御を始めたCPUと、最後にSMCを実行するCPUは区別する。
**PSCI CPU_OFFには停止する別CPUのMPIDRを指定しない。対象CPU自身が呼び出す。**

v2.12.0の通常成功経路の要点：

```text
対象CPUのLinux：スケジューリング・割り込み等の停止準備
  → 対象CPU自身がSMC CPU_OFF
  → psci_do_cpu_off()
      pwr_domain_off_early()：任意。拒否は不可逆処理の前
      親ドメイン取得・ロック・状態協調
      psci_pwrdown_cpu()：アーキテクチャ/キャッシュ停止準備
      pwr_domain_off()：GIC CPU IF停止、SoC停止要求の準備
      汎用処理：ロック解放・PSCI affinity OFF更新・可視性確保
      pwr_domain_pwr_down_wfi()、またはpsci_power_down_wfi()
  → ハードウェア上の安全な停止
```

`pwr_domain_off()`が戻った後にも汎用処理が残る。
**このフックの途中で即時に自己リセット・電源断して後続処理を飛ばさない。**
電源コントローラへ遅延停止を予約する場合も、TF-Aの最終WFI等と同期させる。
最終停止フックがある場合は戻らない。CPU_OFF成功時もLinuxへ戻らない。
参照：[S-OFF](08-references.md)。

### リセットレジスタしか確認されていない段階での未決事項

CPU1〜3の初回起動にリセット解除を使えても、安全な自己停止まで自動的に決まるわけではない。
CPU自身が最終段階でリセット要求できるか、別コントローラが停止を完結するか、
実際に停止したことを何で確認するかをSoC仕様から決める。

PSCIのaffinityがOFFになった時点と物理停止完了は異なる可能性がある。
直後のCPU_ONが、まだ停止途中のCPUを誤って再始動しないよう、HWの完了状態・要求取消・排他を設計する。
単なるWFIループを「電源断済み」「リセット保持済み」と報告しない。
安全な再ONまで確立するまで、未完成のCPU_OFFを利用可能として公開しない。

## 3. CPU0に個別リセット機能がない場合

個別リセットがなくても、電源ドメイン再投入など別の再始動機構があり得る。
次の二条件を分けて確定する。

1. CPU0を単体OFF後に安全に再始動できるか。
2. CPU0を含むシステム全体をリセットまたは電源断できるか。

再始動手段がない条件ではCPU0を単体OFFの対象から外し、終了処理まで稼働させる。
`pwr_domain_off_early()`の`PSCI_E_DENIED`はFWの保護策として使えるが、
Linux側でもホットプラグ停止の初期段階でCPU0を拒否する。
最終CPU_OFF段階の拒否から通常のOnline状態に戻れるとは仮定しない。
採用Linux版のホットプラグ経路を確認して実装位置を決める。

CPU0に代替の再始動機構があれば、その機構とOS側の対応を別途設計・検証した後に対象へ追加する。
「プライマリだからPSCI CPU_OFF不可」というPSCI共通の規則ではない。

## 4. SYSTEM_OFF

通常のLinux稼働後の終了処理はU-Bootを経由しない。
最後に稼働しているCPUから、BL31の別サービスへ入る。
CPU0を残す方針なら、そのCPUで最終処理を実行させるOS設計も確認する。

```text
Linuxの終了処理 → SMC SYSTEM_OFF
  → psci_system_off() → system_off()
  → PMIC/常時給電回路/外部制御回路へ電源断要求
  → 回路がCPU0を含むSoC電源を落とす
```

CPU0を個別リセットしてから電源断する必要はない。
CPU0が要求を書いた後は、CPU0の実行が止まっても制御回路が電源断を完結できる必要がある。
電源制御回路がなければ、実現できるのは「システム停止・通電継続」まで。
`WFI`だけで物理電源OFFになるとは扱わない。

`system_off()`は戻らないコールバック。CPU_OFFのフックを順に呼ぶサービスではない。
参照：[S-SYSTEM](08-references.md)。

## 5. SYSTEM_RESET

```text
Linuxの再起動要求 → SMC SYSTEM_RESET
  → psci_system_reset() → system_reset()
  → CPU0を含む全体リセット / 全体へ作用するWatchdog / 外部回路
  → 所定のシステム初期状態
  → CPU0：Boot ROM等 → BL2 → BL31 cold → U-Boot → Linux
```

CPU0の個別リセットレジスタは不要だが、CPU0に作用する別の全体リセット経路が必要。
CPU1〜3、GIC、DMA、DDR/周辺回路について、リセット範囲と残存状態を定義する。
初期ブートではCPU0だけが実行し、他CPUが古いLinux/BL31のコードを走り続けない契約とする。
CPU用warmベクタ設定が残る場合も、SYSTEM_RESET後には所定のcold経路へ入ることを確認する。
`system_reset()`も戻らない。参照：[S-SYSTEM](08-references.md)。

### 手段別の対応表

| CPU0に作用する全体リセット | 電源制御 | SYSTEM_OFF | 通常の再起動 |
|---|---|---|---|
| あり | 電源断あり | 電源断可能 | 全体リセット可能 |
| あり | なし | 停止・通電継続のみ | 全体リセット可能 |
| なし | 自動OFF→ONあり | 電源断可能 | 外部回路による電源サイクルを検討 |
| なし | 電源断だけあり | 電源断可能 | 自動再投入なしでは不可 |
| なし | なし | 停止・通電継続のみ | 外部操作等が必要 |

これは条件分岐であり、このSoCの実能力を確定した表ではない。
ウォッチドッグも、CPU0を含む必要な範囲を実際にリセットできるものに限る。

## 6. ソフトウェア再起動との区別

BL2へ単に分岐しても、CPU・GIC・DMA・周辺回路がリセット直後の状態になるわけではない。
これをハードウェアリセット相当の完成実装とはしない。
`kexec`はLinuxのみを入れ替える別案で、BL2→BL31→U-Bootを再実行する方式ではない。
採用を決定しておらず、他CPUの安全な停止・デバイス状態・Linux版の条件を別途確認する。

## 7. CPU_SUSPENDとの境界

CPU_OFF後のCPU_ON、システム停止、SYSTEM_RESET、コンテキストを保持/保存するCPU_SUSPENDは別処理。
今回の初期方針は単体起動と停止の整理まで。クラスタ/DDRを停止するサスペンドの詳細は未設計。
