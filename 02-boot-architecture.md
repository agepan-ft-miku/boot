# ブートアーキテクチャ

前提の状態区分は[決定事項](01-decisions.md)に従う。関数の参照先は[ソース一覧](08-references.md)。

## 1. CPU0のcold boot

システム電源投入または所定の全体リセット後、CPU0だけが初期ブートを行う。

```text
CPU0リセット解除 → Boot ROM等 → BL2 (EL3)
  → BL31 cold (EL3) → ERET → U-Boot (NS EL2)
  → Linux primary entry → LinuxのSMP初期化
```

CPU1〜3は、この間リセット保持する。DDR初期化、イメージロード、GIC共有初期化、
BL31の共有データ初期化を後続CPUで繰り返さない。
BL2はBL31とBL33のU-Bootをロードし、BL31へエントリ情報を渡す。
BL31はPSCIを含むEL3ランタイムとしてLinux稼働後も残る。

## 2. CPU1〜3の初回起動・CPUの再起動

```text
稼働CPUのLinux → SMC CPU_ON(target MPIDR, NS entry PA, context ID)
  → 稼働CPUのBL31 → pwr_domain_on(target)
  → 対象CPUのベクタ設定 → リセット解除
  → 対象CPUのBL31 warm → CPU別初期化 → ERET
  → Linux secondary_entry → Online
```

「warm」はそのCPUが過去に動いていたことを条件としない。
システム共有初期化後のCPU追加起動なので、CPU1〜3の初回実行でもwarm経路である。
CPU0の再ONも、実際の再始動機構とOS対応がある場合のみ同じ経路を使う。
CPU0を再起動できない条件では、この経路の対象をCPU1〜3に限定する。

## 3. 二つの入口を混同しない

| 入口 | 値を供給する側 | 使用箇所 |
|---|---|---|
| Secure warm入口 | TF-Aの`plat_setup_psci_ops(sec_entrypoint, ...)` | SoCのリセットベクタ |
| Non-secure入口 | LinuxのCPU_ON引数 | BL31がNSへERETするときの入口 |

リセットベクタは`sec_entrypoint`、通常は`bl31_warm_entrypoint`を指す。
`BL31_BASE`、BL31 cold入口、Linuxの入口、`pwr_domain_on_finish()`の関数アドレスで代用しない。
全CPUが共通のwarm入口を使い、MPIDRを用いてCPU別スタック・データを選択する。
入口のレジスタ表現、アラインメント、上位/下位書き込み順序、ラッチ条件はSoC仕様に従う。

## 4. v2.12のCPU_ON同期

要求元の`psci_cpu_on_start()`は対象CPU用ロックを保持し、状態をON_PENDINGへ設定してから
`pwr_domain_on()`を呼ぶ。成功して戻った後、`cm_init_context_by_index()`でNSコンテキストを準備する。
対象CPUは起動完了処理で同じCPUロックと同期する。[S-ON](08-references.md)

したがって、`pwr_domain_on()`は対象CPUのLinux到達を待たない。
リセット解除後に対象CPUが早く進んでも、汎用PSCIの同期に任せる。
ハードウェア完了待ちを入れる場合はSoC仕様に基づく有限時間の待ちとし、
実行開始済みなのに失敗を返してPSCIだけOFFへ戻る不整合を避ける。

## 5. 共有メモリとCPU別メモリ

| 領域 | 生存期間・扱い |
|---|---|
| BL2の一時領域 | 引き継ぎ完了後の再利用は可能だが、BL31が参照する一時ポインタを残さない |
| BL31 text/data/bss | 全CPUのEL3ランタイムとして常駐 |
| BL31ページテーブル | 後続CPUのwarm起動で利用するため常駐 |
| CPU別スタック・コンテキスト | 4CPU分確保。重複使用させない |
| PSCI状態・ロック・GICドライバ情報 | 全CPU間で整合した状態を保持 |
| U-Boot/Linux/DTB | BL31等の常駐領域と重ならない配置 |

Secureアクセス制御と、Linux/U-Bootへの予約通知は別に実装する。
具体的な番地、サイズ、セキュリティコントローラ設定は未確定。

## 6. Linuxへの入口状態

本案では全CPUのLinux入口をNon-secure AArch64 EL2へ揃える。
CPU0だけU-Bootが設定したCPU別レジスタに依存しないことを確認する。
DAIF、NS側のMMU状態、キャッシュ保守、タイマ周波数・オフセット、
EL3のトラップ設定・割り込みルーティング・コヒーレンシを整合させる。
BL31のEL3用MMUとLinux入口ELのMMUは区別する。
Linuxの採用版は未指定であり、実装時にその版のarm64 boot protocolも確認する。
