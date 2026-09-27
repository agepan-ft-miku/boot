# TF-A v2.12の移植範囲

参照基準はupstream `v2.12.0`。ベンダーSDKのパッチやv2.12系の保守版差分は未確認。
レジスタ番地が未提示なので、本資料は実装分担を定義する。コンパイル可能なSoCポートはまだ作成しない。

## 1. ビルド設定案

AArch64、BL2 EL3直接起動、BL32なしの前提での設定抜粋。

```makefile
RESET_TO_BL2               := 1
RESET_TO_BL31              := 0
PROGRAMMABLE_RESET_ADDRESS := 1
COLD_BOOT_SINGLE_CPU       := 1

HW_ASSISTED_COHERENCY       := 0
WARMBOOT_ENABLE_DCACHE_EARLY := 0

GICV3_SUPPORT_GIC600 := 0
include drivers/arm/gic/v3/gicv3.mk
BL31_SOURCES += ${GICV3_SOURCES} \
                plat/common/plat_gicv3.c
```

`ARCH=aarch64`でビルドし、A53のCPUライブラリを使用する。
実際のプラットフォームファイルやビルド基盤に組み込む際に、ソース重複を確認する。
`COLD_BOOT_SINGLE_CPU=1`はLinuxのCPU数制限ではない。
セカンダリがBL31へ直接入るために`RESET_TO_BL31=1`へ変更しない。

`HW_ASSISTED_COHERENCY=0`は「クラスタ内コヒーレンシを使わない」という意味ではない。
本案ではキャッシュ有効化の順序を従来のPSCI経路へ合わせる。
早期D-cache有効化は安全性確認後の変更候補とし、最初から有効にしない。
参照：[S-BUILD/S-ENTRY/S-GICMK](08-references.md)。

## 2. BL2

| フック・機能 | 内容 |
|---|---|
| `bl2_el3_early_platform_setup()` | 起動引数、初期コンソール、早期SoC設定 |
| `bl2_el3_plat_arch_setup()` | EL3のメモリ属性・変換テーブル・MMU |
| `bl2_platform_setup()` | DDR・クロック・ブート媒体等の共有初期化 |
| I/Oとイメージ記述 | BL31/BL33の格納場所、ロード先、サイズ上限 |
| `bl2_plat_handle_post_image_load()` | BL31へ渡すエントリ情報等 |

`RESET_TO_BL2=1`ではEL3用のフックを使う。
DDRを使う最初のロードより前に`IMAGE_ATTRIB_PLAT_SETUP`経由の初期化が実行されるようにする。
CPU1〜3をこの段階で解除しない。参照：[S-BL2/S-LOAD](08-references.md)。

## 3. BL31 cold

| フック | 内容 |
|---|---|
| `bl31_early_platform_setup2()` | BL2引数を取り込み、BL33の情報とコンソールを準備 |
| `bl31_plat_arch_setup()` | BL31の変換テーブル、MMU、必要な共有コヒーレンシ設定 |
| `bl31_platform_setup()` | GICドライバ/共有部分/CPU0、必要なタイマ等 |
| `plat_setup_psci_ops()` | `sec_entrypoint`を保持し、PSCI操作表を登録 |
| `bl31_plat_get_next_image_ep_info()` | CPU0の次段U-Bootのエントリ情報を返す |

共有GIC初期化はランタイムサービス初期化でのPSCI登録より前に実行される構成。
CPU別の必須初期化をcold専用フックにだけ置かない。参照：[S-BL31](08-references.md)。

## 4. CPU識別とトポロジ

`plat_my_core_pos()`、`plat_core_pos_by_mpidr()`、`plat_get_power_domain_tree_desc()`を実装する。
MPIDRはAff0だけをマスクして済ませず、存在するAffinityの組み合わせを検証する。
CPU内部番号0〜3を、スタック、PSCI、GICR対応、Device Treeで一貫させる。
`plat_my_core_pos()`は早期アセンブリ経路から安全に使えることを確認する。

1クラスタを上位ドメインとして表す場合の例：

```c
#define PLATFORM_CORE_COUNT 4
#define PLAT_MAX_PWR_LVL     1
#define PLAT_NUM_PWR_DOMAINS 5

static const unsigned char power_domain_tree_desc[] = { 1, 4 };
```

これはクラスタ電源断まで実装済みという意味ではない。
当面はCPU0を稼働させ、DDRとクラスタ共有資源を保持する設計。
電源状態定義・協調状態が実際のHW能力に一致することを確認する。

## 5. GIC-500 / GICv3

CPU0のcoldで次の順序を実行する。

```text
gicv3_driver_init(&soc_gic_data)
  → gicv3_distif_init()
  → gicv3_rdistif_init(cpu0_index)
  → gicv3_cpuif_enable(cpu0_index)
```

対象CPUのwarmでは、本案で`pwr_domain_on_finish_late()`から以下だけを行う。

```text
gicv3_rdistif_init(my_index)
  → gicv3_cpuif_enable(my_index)
```

PSCI共通処理がv2.12で自動的にGICを初期化すると仮定しない。
`pwr_domain_on_finish()` → 必要なキャッシュ有効化 → `pwr_domain_on_finish_late()`の順。
空の早期処理でも`pwr_domain_on_finish`の登録を省略しない。
参照：[S-ON/S-GIC/S-SETUP](08-references.md)。

GICD/GICRの番地と範囲、GICRの配置、Affinity対応を確定する。
全GICRをcoldで探索できる場合、`gicr_base`、`rdistif_num=4`、アドレス配列、
`mpidr_to_core_pos`を準備する。各CPUで探索する方式では`gicv3_rdistif_probe()`を
`gicv3_rdistif_init()`より前に実行する。二つの方式を混在させない。

GICv3 System Register InterfaceのCPU別設定、`GICR_WAKER`の遷移、
Secure割り込みとGroup1 Non-secure割り込みの属性を確認する。
CPU単体OFFでDistributor全体を再初期化・停止しない。
参照：[S-GIC/S-ARMGIC](08-references.md)。

## 6. PSCI操作表

| メンバ | 実行CPU・内容 |
|---|---|
| `pwr_domain_on` | 要求元CPU。対象ベクタ設定、リセット解除 |
| `pwr_domain_on_finish` | 起動されたCPU。キャッシュ有効化前のSoC固有処理 |
| `pwr_domain_on_finish_late` | 起動されたCPU。CPU別GIC等 |
| `validate_ns_entrypoint` | 許可したNS実行領域内の入口か検証 |
| `pwr_domain_off_early` | 停止対象CPU。条件によりCPU0停止拒否 |
| `pwr_domain_off` | 停止対象CPU。GIC CPU IF停止、HW停止準備 |
| `pwr_domain_pwr_down_wfi` | 停止対象CPU。必要ならSoC固有の最終停止。戻らない |
| `system_off` | 呼び出した稼働CPU。全体電源制御。戻らない |
| `system_reset` | 呼び出した稼働CPU。全体リセット。戻らない |

未実装操作を「成功するダミー」として公開しない。
非対応の表現とOS側の動作を合わせる。具体的なON/OFF順序は[電源設計](04-power-reset.md)。

## 7. Linux Device Tree方針

各CPUに`enable-method = "psci"`、CPUの`reg`に実際のMPIDR Affinity値を設定する。
PSCIノードは採用TF-Aが対応するcompatibleと`method = "smc"`を使う。
GICノードは`arm,gic-v3`と実メモリマップを設定する。
レジスタ番地未確定のため、仮アドレス入りのDTSを完成品として追加しない。
