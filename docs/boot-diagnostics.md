# GeaconPolaris 起動不良切り分け記録

記録日: 2026-06-05 から 2026-06-07
対象: GeaconPolaris / Polaris_L_UNIFIED / Polaris_R_UNIFIED

## 結論

GeaconPolaris の unified firmware は起動する状態まで復旧した。

最終的に確認できたこと:

- Zephyr deferred-init patch は af6fff80212a の DT_PROP 判定版で起動する。
- Polaris_L_UNIFIED は TPD profile で起動する。
- 保存済み TPD profile が読み込まれる。
- ModuleMux が i2c@40003000 を初期化する。
- Cirque Pinnacle glidepoint_L@2a の初期化に成功する。
- USB endpoint は configured まで進む。
- split central は right peripheral に接続する。
- IQS profile は右手 peripheral から central の `reg 3` input event まで到達し、mouse movement として処理される。

成功ログは docs/logs/2026-06-06-polaris-l-unified-tpd-success.log に保存した。
IQS の成功ログは docs/logs/2026-06-07-iqs-success-central-reg3-input-events.log と docs/logs/2026-06-07-iqs-success-right-reg3-subscribed-log-cap.log に保存した。

## 主因

### 1. Zephyr deferred-init 判定の誤り

最初の deferred-init patch では、base binding に zephyr,deferred-init boolean property を追加したうえで DT_NODE_HAS_PROP を使っていた。

Zephyr の generated header では、未指定ノードにも property の EXISTS 定義が出るため、DT_NODE_HAS_PROP 判定だと GPIO、clock、UART など通常起動に必要な device まで deferred 扱いになった。

実機結果:

- base Zephyr blinky は動作した。
- deferred-init branch の blinky は動作しなかった。
- init_entry の direct init を戻しても動作しなかった。
- iterable section 分離だけでも動作しなかった。
- DT_PROP 判定に変更した blinky / USB CDC は動作した。

判断:

- 主因は DT_NODE_HAS_PROP による全 device deferred 誤判定。
- 修正後は、実際に zephyr,deferred-init = true の device だけ deferred list に入る。

### 2. TPD の gpio-i2c route が不安定だった

TPD profile では、旧構成の gpio-i2c route で Cirque Pinnacle の I2C transaction が失敗していた。

確認した失敗:

~~~text
zmk_input_module: initialized tpd_i2c
pinnacle: i2c read failed on attempt 1: -5
pinnacle: Failed to get the FW ID -5
zmk_input_module: failed to initialize glidepoint_L@2a: -5
~~~

TPD hardware i2c0 diagnostic では、同じ TPD module が hardware i2c0 で初期化できた。

成功ログ:

~~~text
polaris_tpd_hw_i2c_diag: starting delayed TPD hardware i2c probe for glidepoint_L_hw_i2c@2a
pinnacle: X val: 6
pinnacle: Y val: 5
polaris_tpd_hw_i2c_diag: TPD hardware i2c probe initialized glidepoint_L_hw_i2c@2a
polaris_tpd_hw_i2c_diag: TPD hardware i2c device ready: 1
~~~

判断:

- TPD module 本体と Polaris の D4/D5 / DR pin 定義は成立している。
- 問題は TPD module ではなく gpio-i2c route 側にある。
- unified firmware でも TPD は hardware i2c0 route に寄せるのが正しい。

### 3. spi0 と i2c0 の peripheral instance 共有

nRF52840 では spi0 と i2c0 が同じ peripheral instance を共有する。

TPD を hardware i2c0 に戻すには、TB を spi0 から外す必要があった。

対応:

- TPD candidate を hardware i2c0 に移動した。
- TB candidate を spi2 に移動した。
- TPD bus と TB bus はどちらも zephyr,deferred-init のままにした。
- 選択 profile の device だけ ModuleMux が settings 復元後に初期化する。

### 4. settings_reset に Polaris split 設定が漏れていた

Kconfig.defconfig で ZMK_SPLIT の default がグローバルに置かれていた。

そのため settings_reset shield をビルドしたときにも Polaris の split 設定が漏れる可能性があった。

対応:

- ZMK_SPLIT default y は SHIELD_Polaris_L_Base または SHIELD_Polaris_R_Base のときだけに制限した。

### 5. analog input binding が deferred-init property を持てなかった

dya,analog-input.yaml が base.yaml を include していなかったため、DTS に zephyr,deferred-init を書いても generated header に property が出なかった。

結果として JOY 候補の analog_input_0 が未選択でも通常 init され、TPD と共有する D4/D5 に副作用を出す可能性があった。

対応:

- zmk-module-analoginput-rpc 側で dya,analog-input.yaml に base.yaml include を追加した。
- analog_input_0 が deferred device として扱われることを確認した。

## 現在残している実装変更

GeaconPolaris 側:

- ModuleMux は unified firmware の候補 device を定義する。
- TPD は hardware i2c0 を使う。
- TB は spi2 を使う。
- IQS は右手側の排他 module として hardware i2c1 を使う。
- Polaris IQS pin は SDA=D4/P0.04、SCL=D8/P1.13、DR/RDY=D7/P1.12 が正しい。
- build.yaml は unified firmware と通常 settings_reset だけを持つ。

外部 module / Zephyr 側:

- Zephyr fork は zephyr,deferred-init と device_init(dev) を提供する。
- deferred 判定は DT_PROP ベースで行う。
- zmk-input-module は settings 復元後に選択 profile の device だけを初期化する。
- zmk-module-analoginput-rpc は base.yaml include により deferred-init property を受け取れる。
- zmk-module-cirque-trackpad には Pinnacle I2C retry 変更が残っている。

## ENC profile の追加切り分け

2026-06-06 に ENC profile の実機ログを確認した。

確認できたこと:

- 保存済み profile として ENC が読み込まれている。
- ModuleMux は ENC profile を適用している。
- left_encoder_enc の device_init は成功している。
- ZMK の keymap sensor slot 0 は初期化されている。

確認できていないこと:

- EC11 の GPIO edge interrupt が回転時に発火しているか。
- left_encoder_proxy が raw EC11 sensor trigger に確実に接続しているか。
- raw EC11 の rotation value が runtime sensor rotate behavior まで届いているか。

初回対応:

- zmk-input-module の sensor proxy に trigger 再接続処理を追加した。
- sensor proxy は trigger 設定を保持し、active route の sensor が ready になった時点で raw sensor へ接続する。
- trigger 接続成功時に `left_encoder_proxy attached sensor trigger to left_encoder_enc for profile ENC` を出す。
- rotation 正規化時に `left_encoder_proxy normalized left_encoder_enc rotation ...` を debug log として出す。

追加で確認した問題:

- ENC profile に切り替えると実機がフリーズする。
- EC11 は GPIO edge interrupt を使うため、`CONFIG_SENSOR_LOG_LEVEL_DBG=y` を有効にした状態では回転やチャタリングでログが集中し、USB logging 経路を詰まらせる可能性がある。
- DBG log を外した Polaris_L_UNIFIED でも、ENC profile 起動後およそ 1 秒でフリーズする。
- EC11 を回した瞬間にフリーズすることを確認した。

現在の対応:

- Polaris central snippet から `CONFIG_ZMK_INPUT_MODULE_LOG_LEVEL_DBG=y` と `CONFIG_SENSOR_LOG_LEVEL_DBG=y` を外した。
- Polaris_L_UNIFIED を再ビルドし、生成された `.config` で両方が `LOG_LEVEL_DEFAULT` に戻っていることを確認した。
- EC11 trigger を `CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y` から `CONFIG_EC11_TRIGGER_OWN_THREAD=y` に変更した。
- これにより EC11 の trigger handler を system workqueue から分離し、ENC 側の割り込みが ZMK 全体の workqueue を詰まらせているか切り分ける。
- zmk-input-module の sensor proxy で raw EC11 trigger から ZMK sensor handler を直接呼ばないようにした。
- raw EC11 trigger callback は proxy 専用 work を submit するだけにし、proxy work から ZMK sensor handler を呼ぶ。
- これにより EC11 driver の interrupt-disabled section と ZMK の sensor event / behavior 処理を分離する。
- 初回 offload 実装では proxy work 内で `trigger_pending` を `while` 処理していた。
- EC11 のチャタリングで `trigger_pending` が立ち続けると、proxy work が抜けず system workqueue を占有する可能性がある。
- proxy work は 1 回の実行で 1 event だけ処理し、追加 event が残っている場合は work を再 submit する形に変更した。

次の実機確認で見るログ:

~~~text
zmk_input_module: left_encoder_proxy attached sensor trigger to left_encoder_enc for profile ENC
~~~

判断:

- `attached sensor trigger` が出ない場合、proxy の trigger 接続または初期化順序が問題。
- `OWN_THREAD` でも起動後 1 秒程度で固まる場合、EC11 interrupt storm、P1.12/P1.13 の状態、または sensor trigger handler 経路が問題。
- offload 後も回した瞬間に固まる場合、ZMK sensor event 後段、sensor rotate behavior、または keymap の sensor binding が問題。
- bounded proxy work 後も回した瞬間に固まる場合、default layer の `&rsr_msch` を `&inc_dec_kp C_VOL_UP C_VOL_DN` に差し替えた診断版で runtime sensor rotate / mouse scroll 経路を切り分ける。
- `attached sensor trigger` は出て固まらないが HID 挙動がない場合、runtime sensor rotate の binding/settings が問題。

追加診断:

- default layer の `sensor-bindings` を `&rsr_msch` から `&inc_dec_kp C_VOL_UP C_VOL_DN` に一時変更した。
- DTS 上で default layer が `sensor-bindings = < &inc_dec_kp ... >` になっていることを確認した。
- この診断版で回しても固まる場合、runtime sensor rotate / mouse scroll より前の sensor event、behavior queue、EC11/proxy 経路が問題。
- この診断版で固まらない場合、`rsr_msch`、runtime sensor rotate、または mouse scroll binding 後段が問題。

2026-06-07 の確認:

- `inc_dec_kp` 診断版でも、ENC profile でエンコーダを回した瞬間にフリーズする。
- そのため `rsr_msch` / runtime sensor rotate / mouse scroll 後段だけが原因ではない可能性が高い。
- ログ内の `kscan_charlieplex` / `position: 15` は、実機でキーを連打した操作によるものだった。
- したがって、`position: 15` はエンコーダ回転イベントや P1.12/P1.13 由来のキー入力リークとは扱わない。
- `attached sensor trigger` は出ているため、profile 復元、device_init、ZMK sensor slot 初期化、proxy から raw EC11 への trigger 接続までは成立している。

現在の対応:

- zmk-input-module の sensor proxy に `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_TRIGGER_DEBOUNCE_MS` を追加した。
- Polaris central snippet では `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_TRIGGER_DEBOUNCE_MS=2` を設定した。
- raw EC11 trigger callback は 2ms 待ってから proxy work を submit する。
- raw EC11 driver 側では GPIO interrupt が無効化された状態のままなので、機械式エンコーダのバウンスによる連続 trigger をある程度まとめる狙い。
- `Polaris_L_UNIFIED` の `.config` で `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_TRIGGER_DEBOUNCE_MS=2` と `CONFIG_EC11_TRIGGER_OWN_THREAD=y` を確認した。
- default layer の `sensor-bindings` は、切り分け継続のためまだ `&inc_dec_kp C_VOL_UP C_VOL_DN` の診断状態。

次の判断:

- 2ms debounce 版で固まらない場合、主因は EC11 の trigger storm / bounce と判断する。
- 2ms debounce 版でも固まる場合、次は proxy で ZMK sensor handler を呼ばずに event を破棄する診断版を作る。
- event 破棄診断版で固まるなら raw EC11 interrupt / GPIO / driver 側が問題。
- event 破棄診断版で固まらないなら ZMK sensor handler、sensor behavior、または HID event 後段が問題。

2026-06-07 追加確認:

- 2ms debounce 版でも ENC profile でエンコーダを回すとフリーズした。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DROP_TRIGGER_EVENTS` を追加した。
- Polaris central snippet では `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DROP_TRIGGER_EVENTS=y` を有効化した。
- この診断版では raw EC11 trigger は接続したまま、2ms debounce 後に ZMK sensor handler へ渡さず return する。
- この診断版で固まらない場合、EC11 raw driver より後ろの ZMK sensor handler / behavior / HID event 経路が原因。
- この診断版でも固まる場合、EC11 raw driver、GPIO interrupt、またはエンコーダ信号そのものが原因。

2026-06-07 追加確認 2:

- event drop 診断版でも ENC profile でエンコーダを回すとフリーズした。
- この結果から、ZMK sensor handler / sensor behavior / HID event 後段は主因ではない。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_SKIP_TRIGGER_ATTACH` を追加した。
- Polaris central snippet では `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_SKIP_TRIGGER_ATTACH=y` を有効化した。
- この診断版では ZMK sensor trigger 登録は受けるが、raw EC11 sensor への `sensor_trigger_set()` を呼ばない。
- つまり EC11 の A/B GPIO interrupt は有効化されず、エンコーダを回してもENC入力は発生しない。
- この診断版で固まらない場合、`sensor_trigger_set()` 以降の EC11 GPIO interrupt / EC11 trigger thread / 電気的バウンスが原因。
- この診断版でも固まる場合、trigger有効化とは別に、ENC module の信号線またはハードウェア側がMCU動作へ影響している可能性がある。

2026-06-07 追加確認 3:

- skip trigger attach 診断版では、ENC profile でエンコーダを回してもフリーズしなかった。
- したがって、フリーズは `sensor_trigger_set()` で EC11 の A/B GPIO interrupt を有効化した後に発生している。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_SKIP_TRIGGER_ATTACH` は外した。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DROP_TRIGGER_EVENTS=y` は残した。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_TRIGGER_DEBOUNCE_MS` は 2ms から 20ms に増やした。
- この診断版では raw EC11 trigger は接続するが、ZMK sensor handler へは渡さず、EC11 interrupt 再有効化前に20ms待つ。
- 20ms drop 診断版で固まらない場合、EC11 A/B のバウンスまたは再有効化間隔が主因。
- 20ms drop 診断版でも固まる場合、EC11 driver の割り込み構造そのものを変更する必要がある。

2026-06-07 追加確認 4:

- 20ms drop 診断版では、ENC profile でエンコーダを回してもフリーズしなかった。
- したがって、EC11 raw interrupt を有効化すること自体は成立する。
- 主因は、EC11 interrupt 再有効化が速すぎることで、A/B のバウンスを連続して拾い続けることだと判断する。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DROP_TRIGGER_EVENTS` は外した。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_TRIGGER_DEBOUNCE_MS=20` は残した。
- 次の実機確認では、20ms debounce を残したまま ZMK sensor handler へイベントを流し、`inc_dec_kp` の volume up/down が動くかを見る。
- この版で固まらずに volume up/down が動く場合、ENC freeze の対策は 20ms debounce で成立。
- この版で固まらないが volume up/down が動かない場合、EC11/proxy の値変換または `triggers-per-rotation` を見直す。
- この版で再び固まる場合、ZMK sensor handler 後段にも追加の詰まりがある。

2026-06-07 追加確認 5:

- 20ms debounce + 実イベント版では、ENC profile でエンコーダを回すと再びフリーズした。
- `DROP_TRIGGER_EVENTS=y` のときは固まらないため、EC11 raw interrupt の有効化だけではなく、ZMK sensor handler を呼び出す経路も関与している。
- 現在の offload 実装では、raw EC11 trigger handler が proxy work を submit して戻る。
- その直後に EC11 driver が A/B GPIO interrupt を再有効化するため、ZMK sensor handler の `sample_fetch` / `channel_get` / event 処理中にも次のEC11 interruptが入れる。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DIRECT_TRIGGER_HANDLER` を追加した。
- Polaris central snippet では `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DIRECT_TRIGGER_HANDLER=y` を有効化した。
- この診断版では 20ms debounce 後、proxy workへ投げずに、EC11 thread内でZMK sensor handlerを同期実行する。
- これにより、ZMK sensor handler の処理が終わるまで EC11 driver 側のA/B GPIO interrupt再有効化を遅らせる。
- この版で固まらずに volume up/down が動く場合、最終対策は「20ms debounce + 同期forward」。
- この版でも固まる場合、`sample_fetch` / `channel_get` / `raise_zmk_sensor_event` / behavior queue のどこかをさらに分解する。

2026-06-07 追加確認 6:

- 20ms debounce + 同期forward版でも、ENC profile でエンコーダを回すとフリーズした。
- ZMK の sensor event manager は同期実行なので、ZMK sensor handler 内で `sample_fetch` / `channel_get` / `raise_zmk_sensor_event` / keymap behavior 処理まで同じ呼び出しで進む。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_FETCH_ONLY_TRIGGER_EVENTS` を追加した。
- Polaris central snippet では `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_FETCH_ONLY_TRIGGER_EVENTS=y` を有効化した。
- この診断版では raw EC11 trigger 後、20ms debounce してから proxy sensor の `sensor_sample_fetch()` と `sensor_channel_get()` だけを実行する。
- ZMK sensor handler は呼ばず、`raise_zmk_sensor_event` も実行しない。
- この版で固まらない場合、`sample_fetch` / `channel_get` / proxy normalization までは問題なく、`raise_zmk_sensor_event` 以降が原因。
- この版でも固まる場合、EC11 sample fetch、proxy channel get、または rotation normalization が原因。

2026-06-07 追加確認 7:

- fetch-only 診断版でも、ENC profile でエンコーダを回すとフリーズした。
- したがって `raise_zmk_sensor_event`、keymap sensor binding、behavior queue、HID 出力は主因ではない。
- `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_SAMPLE_ONLY_TRIGGER_EVENTS` を追加した。
- Polaris central snippet では `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_SAMPLE_ONLY_TRIGGER_EVENTS=y` を有効化した。
- この診断版では raw EC11 trigger 後、20ms debounce してから proxy sensor の `sensor_sample_fetch()` だけを実行する。
- `sensor_channel_get()` と rotation normalization は実行しない。
- この版で固まらない場合、`sensor_channel_get()` または proxy rotation normalization が原因。
- この版でも固まる場合、EC11 `sample_fetch()`、GPIO read、または EC11 状態更新が原因。

2026-06-07 追加確認 8:

- sample-only 診断版でも、ENC profile でエンコーダを回すとフリーズした。
- したがって `sensor_channel_get()` と proxy rotation normalization も主因ではない。
- 原因範囲は EC11 `sample_fetch()`、A/B GPIO read、EC11 `ab_state/pulses` 更新、またはその直後の割り込み再有効化に絞られた。
- EC11 driver に `CONFIG_EC11_TRIGGER_A_PIN_ONLY` を追加した。
- `CONFIG_EC11_TRIGGER_A_PIN_ONLY=y` のとき、A相だけを GPIO interrupt source として使い、B相は方向判定のための入力読取だけにする。
- Polaris central snippet では `CONFIG_EC11_TRIGGER_A_PIN_ONLY=y` を有効化した。
- sample-only 診断は継続し、A相のみ割り込みで EC11 `sample_fetch()` が固まらないか確認する。
- この版で固まらない場合、A/B両相に `GPIO_INT_EDGE_BOTH` を張ることによる割り込み過多が主因。
- この版でも固まる場合、割り込み源の数ではなく EC11 `sample_fetch()` そのもの、またはA相の電気的状態が原因。

2026-06-07 追加確認 9:

- `CONFIG_EC11_TRIGGER_A_PIN_ONLY=y` と sample-only 診断を組み合わせても、ENC profile でエンコーダを回すとフリーズした。
- これにより、A/B両相を同時に割り込み源にしていることだけでは説明できないことが分かった。
- 原因範囲は、EC11 trigger handler から呼ばれる `sensor_sample_fetch()` の中、特に A/B GPIO read、`ab_state` 更新、`pulses` 更新、またはその後の割り込み再有効化にさらに絞られた。
- 次の診断として EC11 driver に `CONFIG_EC11_SAMPLE_FETCH_READ_ONLY` を追加した。
- `CONFIG_EC11_SAMPLE_FETCH_READ_ONLY=y` では、`sample_fetch()` で A/B GPIO state は読むが、`ab_state` と `pulses` は更新しない。
- `CONFIG_EC11_SAMPLE_FETCH_READ_ONLY=y` でもフリーズした。
- これにより、EC11 state machine の `ab_state/pulses` 更新は主因ではない。
- 直前の drop 診断では固まらなかったため、割り込み再有効化だけでも説明できない。
- 原因範囲は、`sample_fetch()` 内の GPIO read にさらに絞られた。
- 次の診断として `CONFIG_EC11_SAMPLE_FETCH_READ_A_ONLY` と `CONFIG_EC11_SAMPLE_FETCH_READ_B_ONLY` を追加した。
- まず `CONFIG_EC11_SAMPLE_FETCH_READ_A_ONLY=y` で、割り込み元の A 相だけを読んでもフリーズするか確認する。
- `CONFIG_EC11_SAMPLE_FETCH_READ_A_ONLY=y` でもフリーズした。
- これにより、B相 read と EC11 state machine 更新は主因から外れた。
- 原因範囲は、A相 GPIO interrupt 発生後に同じA相 GPIOを読む経路、またはその後の割り込み再有効化に絞られた。
- 次は `CONFIG_EC11_SAMPLE_FETCH_READ_B_ONLY=y` で、A相 GPIO interrupt 発生後にB相だけを読んでも固まるか確認する。
- `CONFIG_EC11_SAMPLE_FETCH_READ_B_ONLY=y` でもフリーズした。
- これにより、A相/B相のどちらを読むかではなく、EC11 GPIO interrupt後にGPIO readを行う組み合わせが問題と判断した。
- 実用修正として、EC11のGPIO割り込み式triggerを使わない `CONFIG_EC11_TRIGGER_POLLING` を追加した。
- `CONFIG_EC11_TRIGGER_POLLING=y` ではGPIO interrupt callbackを登録せず、専用threadが `CONFIG_EC11_POLLING_INTERVAL_MS` 間隔でA/B stateを監視する。
- Polaris central snippet は `CONFIG_EC11_TRIGGER_POLLING=y`、`CONFIG_EC11_POLLING_INTERVAL_MS=2`、`CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DIRECT_TRIGGER_HANDLER=y` に切り替えた。
- sensor proxy は直接ZMK sensor handlerへ渡す。これにより、polling threadで変化検出後、ZMK側の `sample_fetch` / `channel_get` が同期的に走る。

2026-06-07 追加確認 10:

- EC11 polling trigger 版で、フリーズせず sensor event が発火することを確認した。
- ログでは `zmk_keymap_sensor_event` が各 layer に対して呼ばれている。
- ただし `zmk_behavior_sensor_rotate_common_accept_data` の値は `val1=0, val2=0` のままで、キー入力としては発火していなかった。
- 原因候補は2つある。
- 1つ目は、polling thread が状態変化を検出して handler を呼んだあと、ZMK 側の `sample_fetch()` が再度 GPIO を読んでおり、すでに同じ state になっているため delta が0になること。
- 2つ目は、proxy の `triggers-per-rotation=<10>` が EC11 の小さな raw delta を0に丸めていること。
- 対応として、polling mode では polling thread 側で `ec11_update_ab_state()` を呼んで `pulses` を蓄積し、ZMK 側の `sample_fetch()` では GPIO を再読取しないようにした。
- polling thread は delta が0ではないときだけ ZMK handler を呼ぶ。
- ENC profile の proxy `triggers-per-rotation` は `24` に変更した。
- 次回確認では、フリーズしないことに加えて volume up/down 相当の入力が出るかを見る。

2026-06-07 追加確認 11:

- polling thread 側で delta を蓄積する版では、少し入力反応したあとフリーズした。
- これにより、GPIO interrupt は回避できているが、polling thread から ZMK sensor handler を同期実行する経路が重い、またはバウンス由来のイベント量で詰まる可能性が出た。
- 対応として `CONFIG_ZMK_INPUT_MODULE_SENSOR_PROXY_DIRECT_TRIGGER_HANDLER=n` に変更し、EC11 polling thread から ZMK sensor処理を直接呼ばないようにした。
- sensor proxy は通常の `k_work` 経由で ZMK handler を呼ぶ。
- EC11 polling interval は `2ms` から `5ms` に変更した。
- 目的は、機械式エンコーダのバウンスによる過剰なsensor eventを抑え、ZMK側のbehavior queue処理をsystem workへ逃がすこと。
- 次回確認では、フリーズしないこと、volume up/down 相当の入力が出ること、連続回転で詰まらないことを見る。

2026-06-07 追加確認 12:

- EC11 polling trigger + system work forwarding 版で、ENC profile の入力が HID report まで到達することを確認した。
- ログでは `zmk_behavior_sensor_rotate_common_accept_data: val1: 0, val2: 1`、`triggers: 1` が出ている。
- その後 `LAYOUT_SHIFT_KEY_PRESS: 0xc00e9`、`hid_listener_keycode_pressed: usage_page 0x0C keycode 0xE9` が出ている。
- これにより、EC11 polling 側の delta 蓄積、sensor proxy、ZMK sensor behavior、HID report の経路は成立している。
- 残る確認は、長めに回したときにフリーズしないこと、逆方向で `0xC00EA` が出ること、連続回転で behavior queue が詰まらないこと。

2026-06-07 追加確認 13:

- EC11 polling trigger + system work forwarding 版で、左右どちらの回転方向も動作した。
- 一方で、回し続けると過剰に入力される傾向があった。
- 原因は、polling trigger が quadrature の相変化ごとに delta を出しており、1 detent あたり複数回の sensor event が発生していることと判断した。
- 対応として、`left_encoder_enc` の `steps` を `<24>` から `<96>` に変更した。
- proxy 側の `triggers-per-rotation=<24>` は維持する。
- これにより、EC11 driver は1相変化を3.75度として扱い、proxy は4相変化で15度、つまり1 trigger に丸める。
- `steps=<96>` 版では過剰入力は問題なさそうだった。
- これにより、ENC profile は EC11 polling trigger + system work forwarding + `steps=<96>` を実用OK構成として扱う。

2026-06-07 追加確認 14:

- JOY profile が実機で動作することを確認した。
- 詳細なシリアルログは未取得だが、ユーザー観測として JOY は OK。
- これにより、ModuleMux の JOY profile 経路、少なくとも実用上の ADC + encoder 候補経路は成立している。
- 必要になった場合のみ、ADC raw値、JOY側encoder event、input processor 経路を個別ログで確認する。

2026-06-07 追加確認 15:

- TB profile が実機で動作することを確認した。
- 詳細なシリアルログは未取得だが、ユーザー観測として TB は OK。
- 左右どちらのTB、または両方かはこの観測だけでは明記しない。
- これにより、少なくともテストしたTB profileのSPI/PMW3610/入力経路は実用上成立している。
- 必要になった場合のみ、`spi2` 初期化、PMW3610 初期化、input listener / split 経路を個別ログで確認する。

2026-06-07 追加確認 16:

- BT layer の module selection key を押したとき、split 経由の peripheral behavior invocation で behavior label が truncation した。
- ログでは `polaris_module_select` が `polaris_module_` に切り詰められ、さらに Bluetooth split run-behavior payload 側で `polaris_` に切り詰められていた。
- ZMK の Bluetooth split run-behavior payload は behavior device name が 9 bytes、つまり終端NUL込みで最大8文字までになる。
- 対応として、module select behavior の devicetree node name を `pmod` に短縮し、keymap の参照も `&pmod` に変更した。
- これにより、右手側など peripheral 側の module selection key でも truncation なしで behavior を呼び出せる想定。

2026-06-07 追加確認 17:

- `pmod` へ短縮した firmware で BT layer の module selection key を実機確認した。
- ログでは `zmk_keymap_apply_position_state: layer_id: 4 position: 18, binding name: pmod` が出ている。
- その後 `split_bt_invoke_behavior_payload` と `split_central_split_run_callback` に到達している。
- 以前出ていた `Truncated behavior label` は出ていない。
- これにより、module select behavior name の split BT payload 長問題は解消済みとして扱う。

2026-06-07 追加確認 18:

- central serial log だけでは右手側の input module selected/applied profile が分からないことを確認した。
- ログでは central 側の `input module profile loaded: ENC` と split peripheral 接続までは見える。
- 一方で、右手側で `pmod` を実行した後の peripheral 側 selected/applied profile は central log に出ていなかった。
- 原因は、`pmod` behavior が `BEHAVIOR_LOCALITY_EVENT_SOURCE` で peripheral 上実行され、保存自体は peripheral settings に行われるが、その結果を central へ state report していなかったこと。
- 対応として `zmk_input_module_report_state(request_id, status)` を追加した。
- `pmod` behavior は profile 選択後に state report を発行する。
- split peripheral では `zmk_input_module_state_report` が central へ relay される。
- split central では受け取った report を次の形式でログ出力する。
- さらに split peripheral が central へ接続した後、1000ms 遅延して state report を自動送信するようにした。
- これにより、boot 後に右手側が接続された時点でも右手側の selected/applied profile を central serial log から確認できる。

~~~text
input module state report from source=1 request=0 selected=TPD (4) applied=ENC (1) applied_now=yes status=0
~~~

- `source=0` は central/local。
- `source=1` は最初の split peripheral slot で、Polaris 構成では右手側として扱う。
- `selected` はその半身に保存された次回起動用 profile。
- `applied` はその半身で現在初期化済みの profile。
- `selected` と `applied` が違う場合、その半身は再起動後に新しい module profile が有効になる。

2026-06-07 追加確認 19:

- 右手側 profile を変更したときに freeze した。
- 直前の実装では、右手側 `pmod` が split run-behavior 経由で peripheral 上実行され、その callback 内で `zmk_input_module_select_set()` と state report relay を直接実行していた。
- `zmk_input_module_select_set()` は `settings_save_one()` を呼ぶため、split BLE/GATT の behavior callback 経路で flash settings 書き込みと relay notify が走る可能性があった。
- 対応として、keymap behavior は profile 選択を直接実行せず、zmk-input-module 専用 workqueue に積むだけに変更した。
- 実際の profile 選択、settings 保存、state report は専用 workqueue から実行する。
- Studio/WebUI から右手側へ転送される select request も同様に専用 workqueue 経由にした。
- 専用 workqueue は `CONFIG_ZMK_INPUT_MODULE_WORK_QUEUE_STACK_SIZE=1024` / `CONFIG_ZMK_INPUT_MODULE_WORK_QUEUE_PRIORITY=10` で構成する。
- state report は `CONFIG_ZMK_INPUT_MODULE_STATE_REPORT_BEHAVIOR_DELAY_MS=50` により 50ms 遅延して送る。
- 接続後の自動 state report は、右手側が無線接続した瞬間に freeze する切り分けのため、デフォルトでは無効化する。
- `CONFIG_ZMK_INPUT_MODULE_STATE_REPORT_CONNECT_DELAY_MS=0` の場合、split peripheral 接続時には state report を送らない。
- 接続後に右手側情報が必要な場合は、profile 選択後の report または Studio/WebUI からの明示的な state request で確認する。
- この版で右手側 profile 変更時に freeze しなければ、callback-path 内または system workqueue 上での settings 保存または relay notify が原因だったと判断する。

2026-06-07 追加確認 20:

- 右手 profile 変更時ではなく、右手側が無線接続した瞬間に freeze した。
- 直前の版では、split peripheral 接続イベントで自動 state report を予約していた。
- 対応として、接続時の自動 state report をデフォルト無効化した。
- この版で接続時 freeze が消えれば、接続直後の peripheral-to-central relay event が原因候補になる。
- この版でも接続時 freeze する場合、state report ではなく split relay event の登録、DYA Studio RPC、または右手側初期化処理を次に切り分ける。

2026-06-07 追加確認 21:

- 接続時自動 state report を無効化しても、右手側が無線接続した後に freeze した。
- ログは `Found run behavior handle` の後、relay event characteristic の subscribe 相当箇所で止まっている。
- `zmk-input-module` の `select ZMK_SPLIT_RELAY_EVENT if ZMK_SPLIT` を削除し、Polaris L/R の `CONFIG_ZMK_SPLIT_RELAY_EVENT=y` も削除した。
- ただし `INPUT_PINNACLE_STUDIO_RPC` と `INPUT_IQS9151_STUDIO_RPC` も `ZMK_SPLIT_RELAY_EVENT` を強制有効化していたため、Polaris設定側でこの2つのStudio RPCも無効化した。
- 外部モジュール本体は変更しない。
- 目的は、relay event characteristic 自体をGATT serviceから外し、接続時 freeze が消えるか確認すること。
- この版では DYA Studio から右手側 profile / TPD / IQS を読む/書くsplit RPC機能は無効になる。
- 右手側のキーで `&pmod` を押して profile を保存する経路は split run-behavior なので残る。

2026-06-07 追加確認 22:

- ログを再確認すると、停止位置の `handle 35` は relay event ではなく sensor state characteristic の subscribe である可能性が高い。
- ZMK に `CONFIG_ZMK_SPLIT_BLE_SENSOR_STATE` を追加し、split BLE の sensor state characteristic / subscribe / notify だけを無効化できるようにした。
- Polaris L/R では `CONFIG_ZMK_SPLIT_BLE_SENSOR_STATE=n` とした。
- 左手ローカルENCなどの通常の `zmk_sensor_event` 処理は残る。無効化されるのは split BLE 経由の sensor notification のみ。
- この版で右手接続時 freeze が消えれば、原因は split sensor state characteristic の subscribe/notify 経路に絞れる。

2026-06-07 追加確認 23:

- `CONFIG_ZMK_SPLIT_BLE_SENSOR_STATE=n` かつ右手側 split RPC 無効化後、右手側の key position notification は central へ届いた。
- 左手position 12でlayer 4へ入り、右手position 18がlayer 4の `pmod` として解決された。
- central側では `split_bt_invoke_behavior_payload` と `split_central_split_run_callback` をpress/release両方で確認した。
- この結果から、右手側key position経路と `pmod` のsplit run-behavior呼び出しは成立している。
- この診断版では split relay event と右手側Studio split RPCを無効化しているため、central serial logだけでは右手側profileの保存結果は確認できない。
- 次の確認は、右手側シリアルログで `input module select behavior pressed` / `input module profile saved` を見ること、または右手再起動後に選択profileが適用されることを確認すること。

2026-06-07 追加確認 24:

- 右手側シリアルログが無効になっていた。
- 原因は `Polaris_R_UNIFIED` target から `zmk-usb-logging` snippet が外れていたこと。
- `CONFIG_ZMK_USB_LOGGING=y` だけでは CDC ACM UART の devicetree node が足りず、Kconfig warning でbuildが失敗する。
- 対応として、`build.yaml` の `Polaris_R_UNIFIED` snippet に `zmk-usb-logging` を戻した。
- `CONFIG_ZMK_USB` は split peripheral では依存関係により無効化されるが、`CONFIG_ZMK_USB_LOGGING=y` / `CONFIG_LOG=y` / `CONFIG_CONSOLE=y` / `CONFIG_USB_CDC_ACM=y` は有効になった。
- `Polaris_R_UNIFIED.uf2` は再ビルド済み。

2026-06-07 追加確認 25:

- 右手側シリアルログで `pmod` が peripheral 側まで届いていることを確認した。
- `pmod POLARIS_MODULE_TB` は `input module profile already TB` になった。
- `pmod POLARIS_MODULE_TPD` は `input module profile selected for next boot: TPD` / `input module profile saved: TPD` まで到達した。
- `pmod POLARIS_MODULE_IQS` も `input module profile selected for next boot: IQS` / `input module profile saved: IQS` まで到達した。
- 再起動後、右手側は settings から `IQS` を読み込み、`iqs_i2c` を初期化した。
- その後 `iqs9151@56` の runtime config apply で `Failed to apply rotate settings (-5)` / `runtime config apply failed: -5` となった。
- この結果から、右手側 profile 選択、split run-behavior、settings 保存/復元、USB serial logging は成立している。
- 残件は IQS profile 適用後の IQS9151 device init であり、ModuleMux の profile 選択問題とは分けて扱う。
- 詳細ログは `docs/logs/2026-06-07-right-profile-save-iqs-init-fail.log` に保存した。

2026-06-07 追加対応 26:

- IQS9151 driver の runtime config 適用時に、各 I2C read/write 前へ RDY wait を追加した。
- 対象は `iqs9151_update_bits_u16()`、runtime config の X/Y resolution、ATI target、dynamic filter、bottom beta 書き込み。
- 失敗時に register address と read/write 種別がわかる error log も追加した。
- 目的は、初期 bulk config 後に `Failed to apply rotate settings (-5)` で落ちるケースを、device ready handshake 不足として切り分けること。
- `./just build Polaris_R_UNIFIED` は成功した。
- 更新後の `firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2` は 518656 bytes。
- 次の実機確認では、IQS profile 起動時に runtime config が通るか、または `I2C read/write 0x.... failed` の詳細ログが出るかを見る。

2026-06-07 追加確認 27:

- 追加対応 26 後の実機ログでは、右手側は settings から `IQS` を読み込み、`iqs_i2c` を初期化した。
- その後 `iqs9151: RDY timeout after 500ms` が出て、`iqs9151@56` 初期化が `-EIO` で失敗した。
- 今回は runtime config apply まで進まず、IQS9151 の ready/product-read 段階で止まっている。
- `iqs_i2c` は初期化済みなので、ModuleMux の profile 選択と software I2C bus 初期化は成立している。
- IQS9151 driver README の例も `irq-gpios = <... GPIO_ACTIVE_LOW>` なので、DTS 上の active-low 指定は標準例と一致している。
- 次の対応として、RDY timeout log に logical/raw/flags を追加し、product number read を retry するようにした。
- `./just build Polaris_R_UNIFIED` は成功した。
- 更新後の `firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2` は 519168 bytes。
- 詳細ログは `docs/logs/2026-06-07-iqs-rdy-timeout-product-read-fail.log` に保存した。

2026-06-07 追加確認 28:

- RDY/product-read retry 対応後の実機ログでは、IQS RDY timeout が 5 回発生した。
- 各 timeout は `logical=0 raw=1 flags=0x0011`。
- `flags=0x0011` は active-low + pull-up なので、DR/RDY 物理線は High のまま。active-low の ready は出ていない。
- product number read は 5 回すべて `-EIO`。
- つまり `iqs_i2c` bus は初期化されているが、configured bus 上の IQS9151 `0x56` から ACK が取れていない。
- 生成 DTS 上の Polaris IQS pin は SDA=D4/P0.04、SCL=D8/P1.13、DR/RDY=D7/P1.12。
- `uart0` は生成 DTS で `status = "disabled"` のため、D7/RX が UART に保持されている可能性は低い。
- Polaris IQS pin は SDA=D4/P0.04、SCL=D8/P1.13、DR/RDY=D7/P1.12 が正しい。
- 旧 `Polaris_R_IQS.overlay` も同じ pin を hardware `i2c0` で使っていた。
- したがって pin mapping は原因候補から外した。この時点では、次の firmware-side の差分を unified 実装で IQS を `gpio-i2c` にしている点として扱った。
- 詳細ログは `docs/logs/2026-06-07-iqs-rdy-high-product-read-retry-fail.log` に保存した。

2026-06-07 追加対応 29:

- Polaris IQS pin は、ユーザー確認により SDA=D4/P0.04、SCL=D8/P1.13、DR/RDY=D7/P1.12 を正とした。
- 旧 `Polaris_R_IQS.overlay` も同じ pin を hardware `i2c0` で使っていた。
- unified firmware では TPD が hardware `i2c0` を使うため、IQS は `gpio-i2c` ではなく deferred hardware `i2c1` に移した。
- `snippets/ModuleMux/ModuleMux.overlay` の右手側 `profile_iqs` は `devices = <&i2c1 &iqs9151>;` になった。
- `snippets/Peripheral/Peripheral.conf` から `CONFIG_I2C_GPIO` は外した。
- `./just build Polaris_R_UNIFIED` は成功した。
- 更新後の `firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2` は 516608 bytes、SHA256 は `b1290015b45609d903383c3ff68b0d023604bf931516766ef5ca7e203eb6c6b4`。
- 生成 DTS では `i2c@40004000` が `zephyr,deferred-init` 付きで有効化され、子 node として `iqs9151@56` が定義されている。
- 次の実機確認では、右手側 IQS profile 起動時に `initialized i2c@40004000` と `initialized iqs9151@56` まで進むか、または新しい失敗箇所を確認する。

2026-06-07 追加確認 30:

- hardware i2c1 化後の実機ログで、右手側は settings から `IQS` profile を読み込んだ。
- ModuleMux は deferred hardware `i2c1` を `i2c@40004000` として初期化した。
- IQS9151 は RDY timeout を 3 回出したが、その後 `iqs9151@56` の初期化に成功した。
- 重要行:

~~~text
zmk_input_module: input module profile loaded: IQS
zmk_input_module: input module profile=IQS kscan=disabled encoder=disabled adc=disabled spi=disabled i2c=enabled
zmk_input_module: initialized i2c@40004000
iqs9151: RDY timeout after 500ms (logical=0 raw=1 flags=0x0011)
iqs9151: RDY timeout after 500ms (logical=0 raw=1 flags=0x0011)
iqs9151: RDY timeout after 500ms (logical=0 raw=1 flags=0x0011)
zmk_input_module: initialized iqs9151@56
~~~

- これにより、Polaris IQS の SDA=D4/P0.04、SCL=D8/P1.13、DR/RDY=D7/P1.12 と hardware `i2c1` route は成立した。
- `gpio-i2c` 版の product number read `-EIO` は、pin mapping ではなく `gpio-i2c` route または timing の問題として扱う。
- RDY timeout が 3 回出るため、起動時の IQS 初期化待ちは約 3 秒ある。現状は成功しているため機能上は許容できるが、UX として短縮したい場合は IQS9151 driver の power-on delay / retry interval を別途調整する。
- 詳細ログは `docs/logs/2026-06-07-iqs-hardware-i2c1-init-success.log` に保存した。

2026-06-07 追加確認 31:

- `i2c@40004000` と `iqs9151@56` の初期化は成功したが、IQS 自体の入力は動作していない。
- central 側には `Polaris_L_interior.dsti` 由来で `trackpad_split@3` と `trackpad_listener` が存在するため、reg `3` の受け口はある。
- したがって次の切り分けは、右手側で IQS9151 の DR/RDY interrupt と frame read が発生しているかを見る。
- IQS9151 driver に先頭 16 件だけの診断ログを追加した。
- 見るログ:

~~~text
iqs9151: interrupt enabled logical=... raw=...
iqs9151: irq #...
iqs9151: work #... reading frame
iqs9151: frame #... rel=(...) info=... tp=... fingers=...
iqs9151: report #... rel/key ...
~~~

- `irq #...` が出ない場合は、DR/RDY pin、event mode、IQS9151 側の touch detection / ATI / config を疑う。
- `frame #...` は出るが `report #...` が出ない場合は、frame の finger/movement flags が cursor/scroll 条件を満たしていない。
- `report #...` が出るが central 側で動かない場合は、`iqs_input_proxy_r`、`input-split` reg `3`、central `trackpad_listener` の転送経路を疑う。
- 診断ログ追加後の `./just build Polaris_R_UNIFIED` は成功した。
- 更新後の `firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2` は 518144 bytes、SHA256 は `a246f7021cd1bc2855bb24dd495b4a88e40e4c25516517615d7159c86a368168`。

2026-06-07 追加確認 32:

- 診断版の実機ログで、IQS を触ったときに `irq #... pins=0x00001000 logical=1 raw=0` が連続して出ることを確認した。
- つまり DR/RDY は touch 操作で active-low に落ちており、IRQ pin と event mode は動作している。
- 一方で `work #... reading frame` が出ていないため、GPIO callback から submit した frame read work が実行されていない。
- IQS9151 driver を system workqueue 依存から外し、専用の `IQS9151 Work Queue` に `k_work_submit_to_queue()` するように変更した。
- GPIO callback のログに `submit=...` も出すようにした。
- 次の実機確認では、以下を見る。

~~~text
iqs9151: irq #... logical=1 raw=0 submit=...
iqs9151: work #... reading frame
iqs9151: frame #... rel=(...) info=... tp=... fingers=...
iqs9151: report #... rel/key ...
~~~

- `submit=...` が正常値なのに `work #...` が出ない場合は、専用 workqueue 起動または priority/stack を疑う。
- `work #...` と `frame #...` が出る場合は、次に `report #...` の有無で gesture/frame 判定へ進む。
- 専用 workqueue 化後の `./just build Polaris_R_UNIFIED` は成功した。
- 更新後の `firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2` は 518656 bytes、SHA256 は `ce7204edc3ec296202511770c3aaae8f2c4db8938edd9f32344b8ff8025ad501`。

2026-06-07 追加確認 33:

- 専用 workqueue 化後の実機ログで、IQS は `irq`、`work`、`frame`、`report` まで到達した。
- 重要行:

~~~text
iqs9151: irq #0 pins=0x00001000 logical=1 raw=0 submit=1
iqs9151: work #0 reading frame
iqs9151: frame #0 rel=(0,0) info=0x2210 tp=0x7f41 fingers=1 ...
iqs9151: frame #12 rel=(-22,35) info=0x2200 tp=0x7f11 fingers=1 ...
iqs9151: report #0 rel code=0x0000 value=-22 sync=0
iqs9151: report #1 rel code=0x0001 value=35 sync=1
~~~

- これにより、IQS9151 driver は touch を検出し、frame を読み、relative input event を生成できている。
- その後に freeze したため、残件は IQS9151 の sensing ではなく、split BLE input notification の過負荷または優先度問題として扱う。
- 現状の split input は X/Y を別々の `bt_gatt_notify()` として送るため、10ms sampling では 1 秒あたり最大 200 件近い通知になる。
- 対応として、IQS9151 work queue priority を `0` から `9` に下げ、BT/HCI thread を優先できるようにした。
- さらに IQS9151 の `ACTIVE_MODE_SAMPLING_PERIOD_0` を `0x0A` から `0x1E` に変更し、active sampling を約 30ms に落とした。
- 目的は split BLE notification rate を下げ、右手 peripheral 全体が固まることを避けること。
- 変更後の `./just build Polaris_R_UNIFIED` は成功した。
- 更新後の `firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2` は 518656 bytes、SHA256 は `4ad29e789e068843d42c7263b445b31f419c40e5ee22f49d74f2593018946c99`。

## 整理した診断用ファイル

起動問題の解消後、以下の診断足場はリポジトリから削除した。

- config/PolarisDiag.keymap
- snippets/CentralDiag/
- snippets/NoInputModule/
- snippets/NoSplitDiag/
- snippets/NoStudioNoOled/
- snippets/TpdHwI2cDiag/
- snippets/UsbOnlyDiag/
- src/tpd_hw_i2c_diag.c
- build.yaml の Polaris_L_DIAG_* target
- build.yaml の settings_reset_LOG target

firmware/zmk-config-GeaconPolaris には最終成果物だけを残した。

- Polaris_L_UNIFIED.uf2
- Polaris_R_UNIFIED.uf2
- settings_reset-seeeduino_xiao_ble.uf2

## 保存しているログ

詳細ログは docs/logs/ に保存している。

- 2026-06-06-tpd-100khz-retry-read-fail.log
- 2026-06-06-tpd-hw-i2c-diag-no-pinnacle-log.log
- 2026-06-06-tpd-hw-i2c-diag-success.log
- 2026-06-06-polaris-l-unified-tpd-success.log
- 2026-06-06-enc-startup-no-rotation-events.log
- 2026-06-07-enc-freeze-after-inc-dec-diagnostic.log
- 2026-06-07-enc-polling-events-zero-value.log
- 2026-06-07-enc-polling-system-work-volume-ok.log
- 2026-06-07-enc-polling-bidir-ok-overinput.log
- 2026-06-07-enc-steps96-ok.log
- 2026-06-07-joy-profile-ok.log
- 2026-06-07-tb-profile-ok.log
- 2026-06-07-tb-profile-init-ok-disconnected-enotconn.log
- 2026-06-07-module-select-label-truncated.log
- 2026-06-07-module-select-pmod-no-truncation.log
- 2026-06-07-right-module-state-not-visible-before-report.log
- 2026-06-07-right-profile-change-freeze-async-fix.log
- 2026-06-07-right-profile-save-iqs-init-fail.log
- 2026-06-07-right-pmod-split-run-ok-sensor-state-disabled.log
- 2026-06-07-iqs-rdy-timeout-product-read-fail.log
- 2026-06-07-iqs-rdy-high-product-read-retry-fail.log
- 2026-06-07-iqs-hardware-i2c1-init-success.log
- 2026-06-07-iqs-input-report-freeze-before-split-queue.log
- 2026-06-07-iqs-split-input-notify-enotconn.log
- 2026-06-07-iqs-reports-after-disconnect-enotconn-reg3.log
- 2026-06-07-iqs-reg3-not-subscribed-only-reg0.log
- 2026-06-07-iqs-reg3-drop-central-subscribed-reg0.log
- 2026-06-07-iqs-central-cpf-mismatch-reg4-value36.log
- 2026-06-07-iqs-reg3-still-not-subscribed-right-only.log
- 2026-06-07-iqs-central-only-reg0-before-direct-cpf-read.log
- 2026-06-07-iqs-success-central-reg3-input-events.log
- 2026-06-07-iqs-success-right-reg3-subscribed-log-cap.log

各ログの意味は docs/logs/README.md にまとめた。

## 最終確認済み

ビルド:

- Polaris_L_UNIFIED は build 成功。
- Polaris_R_UNIFIED は build 成功。2026-06-07 の IQS runtime config RDY wait 対応後、RDY/product-read retry 対応後、IQS hardware i2c1 化後も build 成功。
- settings_reset-seeeduino_xiao_ble は成果物として保持。

実機:

- Polaris_L_UNIFIED は TPD profile で起動成功。
- hardware i2c0 上の glidepoint_L@2a 初期化成功。
- USB endpoint configured。
- split central 接続成功。
- JOY profile は実機で動作確認済み。
- TB profile は実機で動作確認済み。左右個別の詳細ログは未取得。
- 右手側 TB profile は `spi@40023000`、`trackball@0`、PMW3610 の初期化成功をログで確認済み。
- 右手側 TB profile は実機でカーソルが動くことを確認済み。これにより split input `reg 0` の peripheral notify、central receive、central listener 経路は成立している。
- ENC profile は実機で両方向動作確認済み。`steps=<96>` 版で過剰入力も問題なさそう。
- 右手側 IQS profile は deferred hardware i2c1 で `i2c@40004000` と `iqs9151@56` の初期化成功を確認済み。ただし起動時に RDY timeout retry が 3 回発生する。
- 右手側 IQS profile は IRQ、frame read、input report 発行まで確認済み。
- 右手側 IQS profile は `reg 3` の input report が central に届き、mouse movement として処理されることを確認済み。
- 右手側 IQS profile のログが `irq #15` / `report #15` 以降で止まって見える件は、central 側の入力継続と実機動作から、フリーズではなく診断ログ上限によるものと判断する。

## 2026-06-07 追加確認 34: IQS input report 後 freeze への対応

添付ログ `2026-06-07-iqs-input-report-freeze-before-split-queue.log` では、右手側 IQS profile で以下を確認した。

- `zmk_input_module: initialized i2c@40004000`
- `zmk_input_module: initialized iqs9151@56`
- `iqs9151: irq #...`
- `iqs9151: work #... reading frame`
- `iqs9151: frame #...`
- `iqs9151: report #...`

このため、IQS9151 の初期化、DR/RDY interrupt、I2C frame read、input report 発行までは成立している。
一方で、その後に右手 peripheral が freeze するため、残る主な疑いは split BLE の input notification 経路。

ZMK の `app/src/split/bluetooth/service.c` を確認すると、position、sensor、relay は queue と `service_work_q` を経由して通知している。
しかし `CONFIG_ZMK_INPUT_SPLIT` の input event は、input path から `bt_gatt_notify()` を直接呼んでいた。
IQS は短時間に相対座標 report を連続発行するため、この同期 notify が BLE 側や caller thread を詰まらせる可能性がある。

対応として、input split event も他の split event と同じように bounded message queue と `service_work_q` 経由に変更した。

- `split_input_event_msg` を追加。
- `input_event_msgq` を追加。
- `zmk_split_bt_report_input()` は `K_NO_WAIT` で queue に積むだけに変更。
- queue が満杯の場合は最古の input event を 1 件捨てて最新 event を入れる。
- 実際の `bt_gatt_notify()` は `service_input_event_notify_work` から実行する。

この変更後の `./just build Polaris_R_UNIFIED` は成功した。

- artifact: `firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2`
- size: 519680 bytes
- SHA256: `69ac3741f9c92f272e2f48c8901b9821fe6ac8d37b5dc819e1c184d71872ca3f`

次の実機確認では、この artifact を右手に書き込み、IQS 操作時に freeze が消えるかを見る。
freeze が消えて動きが途切れる場合は、queue size、coalescing、report rate の調整に進む。
まだ freeze する場合は、central 側の input split receive path、または BLE notify callback 周辺を次の切り分け対象にする。

## 2026-06-07 追加確認 35: split input notify `-ENOTCONN` への対応

添付ログ `2026-06-07-iqs-split-input-notify-enotconn.log` では、前回の非同期 notify 化後も右手側 IQS profile で input report までは出ている。

重要行:

- `iqs9151: report #0 rel code=0x0000 value=-24 sync=0`
- `zmk: send_input_event_callback: Error notifying input split event -128`
- `iqs9151: report #1 rel code=0x0001 value=72 sync=1`
- `zmk: send_input_event_callback: Error notifying input split event -128`

`-128` は Zephyr の `ENOTCONN`。
`bt_gatt_notify(NULL, ...)` では、対象 characteristic を購読している接続が見つからない場合にこの値が返る。
つまり IQS driver、input report、queue 化までは成立しているが、central 側が右手 peripheral の IQS input characteristic を正しく subscribe できていない可能性が高い。

確認した central 側 DTS では、input-split reg は以下の 4 つ。

- `trackball_split@0`
- `stick_split@2`
- `trackpad_split@3`
- `tpd_split@4`

IQS は右手側 `trackpad_split@3` として通知するため、reg `3` 自体は central 側にも存在する。
ただし ZMK central 側の subscription 管理で、`reg == 0` を「reg 未設定」と同じ扱いにしていた。
Polaris は `trackball_split@0` を使うため、最初の input slot が永遠に pending 扱いになり、その後の input characteristic discovery / subscription が崩れる可能性がある。

対応:

- `struct peripheral_input_slot` に `reg_set` を追加。
- pending 判定を `!slot->reg` ではなく `!slot->reg_set` に変更。
- CPF から reg を読めた時点で `reg_set = true` にする。
- disconnect / release 時に `reg_set`、handle、notify callback を明示的に初期化。
- central 側で `Input split reg ... value_handle ... ccc_handle ...` をログ出力する。
- peripheral 側 notify error も `reg` 付きでログ出力する。

この修正後、左右を再ビルドした。

- `Polaris_L_UNIFIED.uf2`: 1012224 bytes, SHA256 `06417d08459be695c4783b0da78468515417e1e91dd7ce976f5ab680cf5f9334`
- `Polaris_R_UNIFIED.uf2`: 519680 bytes, SHA256 `fcb94884cbe9b97266bdd71da766399e8cdb2f06d70d853d0fcd0826d13fbb25`

この修正は central 側 subscription 管理に入っているため、右手だけでなく左手 `Polaris_L_UNIFIED` も書き換える必要がある。
次の実機確認では、central serial log に以下が出ることを見る。

- `Input split reg 0 ...`
- `Input split reg 3 ...`
- `Input split reg 4 ...`

右手側では、IQS 操作時に `Error notifying input split event reg 3: -128` が消えるかを見る。

## 2026-06-07 追加確認 36: 右手 TB profile の初期化確認

添付ログ `2026-06-07-tb-profile-init-ok-disconnected-enotconn.log` では、右手側 profile を TB に変更し、再起動後に TB profile が復元されている。

重要行:

- `zmk_input_module: input module profile saved: TB`
- `zmk_input_module: input module profile loaded: TB`
- `zmk_input_module: input module profile=TB kscan=disabled encoder=disabled adc=disabled spi=enabled i2c=disabled`
- `zmk_input_module: initialized spi@40023000`
- `zmk_input_module: initialized trackball@0`
- `pmw3610: PMW3610 initialized successfully`

このため、右手側 TB profile の ModuleMux、SPI bus、PMW3610 device init は成立している。

後半には `split_input_events_ccc: value 0` と `Disconnected ... reason 0x08` が出ており、その後に TB 入力イベントが出て `bt_gatt_notify()` が `-128` (`ENOTCONN`) を返している。
これは未接続状態で input split notify を送ろうとした結果として扱う。
このログでは notify error が `reg` なし形式なので、`Error notifying input split event reg ...` を追加した後の最新右手 artifact ではない可能性がある。

その後、右手側 TB profile で実際にトラックボール入力が動作したことを確認した。
これにより、右手 peripheral から central への input split 基盤そのものは成立している。
残件は IQS の `reg 3` 固有の送受信、または central 側 `trackpad_listener` の処理に絞る。

## 2026-06-07 追加確認 37: IQS report は出るが操作時点で切断済み

添付ログ `2026-06-07-iqs-reports-after-disconnect-enotconn-reg3.log` では、右手側 IQS profile で以下を確認した。

- `zmk_input_module: input module profile loaded: IQS`
- `zmk_input_module: initialized i2c@40004000`
- `zmk_input_module: initialized iqs9151@56`
- `split_input_events_ccc: value 1`
- `Peripheral connected, blinking blue`
- `split_input_events_ccc: value 0`
- `disconnected: Disconnected ... reason 0x08`
- `iqs9151: report #...`
- `send_input_event_callback: Error notifying input split event reg 3: -128`

重要なのは、IQS の `irq`、`frame`、`report` が出る前に central との接続が切れていること。
したがって、このログの `reg 3: -128` は「central が `reg 3` を購読できていない」だけではなく、「そもそも操作時点で接続がない」状態として扱う。
IQS driver 側は入力を生成できている。

対応として、peripheral 側の input split service に reg ごとの購読状態を持たせた。

- input-split CCC callback で `reg` と subscription 状態を追跡する。
- `reg` が購読されていない場合、`bt_gatt_notify()` まで進めず input event を drop する。
- drop log は上限付きにして、未接続時にログが埋まらないようにする。

この変更後の `Polaris_R_UNIFIED.uf2` は build 成功。

- size: 520192 bytes
- SHA256: `8d78f754a7266339cf3b6b83de8894ae827674797af21acb0aa592a0a764eaba`

次の切り分けでは、右手ではなく左手 central 側ログを見る。
特に、IQS profile 接続時に以下が出るかを確認する。

- `Input split reg 3 ...`
- `Got peripheral event for 3!`
- `Disconnected ... reason 0x08` の直前のログ

## 2026-06-07 追加確認 38: IQS は `reg 3` が subscribe されていない

添付ログ `2026-06-07-iqs-reg3-not-subscribed-only-reg0.log` では、右手側に以下が出ている。

- `split_input_events_ccc: reg 0 value 1 subscribed 1`
- `iqs9151: report #0 rel code=0x0000 value=-13 sync=0`
- `zmk_split_bt_report_input: Dropping input split event reg 3; no subscriber`
- `iqs9151: report #1 rel code=0x0001 value=-43 sync=1`
- `zmk_split_bt_report_input: Dropping input split event reg 3; no subscriber`

このログでは、central は右手の `reg 0` だけを subscribe している。
右手 TB は `reg 0` を使うので動作する。
右手 IQS は `reg 3` を使うので、右手側で正しく drop されている。

したがって、現時点の結論は以下。

- IQS9151 hardware init は成功している。
- IQS9151 は IRQ、frame、input report を生成している。
- TB が動作するため、split input 基盤と `reg 0` 経路は成立している。
- IQS が動かない直接原因は、central が `reg 3` を subscribe していないこと。

次に必要な確認は左手 central 側。

- 左手が `Polaris_L_UNIFIED.uf2` の最新 artifact で書き込まれていること。
- central serial log に `Input split reg 0 ...` だけでなく `Input split reg 3 ...` が出ること。
- `Input split reg 3 ...` が出ない場合、central の input characteristic discovery が最初の input characteristic で止まっている。
- `Input split reg 3 ...` が出るのに右手側で `reg 3 subscribed` が出ない場合、GATT subscribe/write が失敗している。

## 2026-06-07 追加確認 39: central の CPF 対応付けずれを確認

添付ログ `2026-06-07-iqs-reg3-drop-central-subscribed-reg0.log` と `2026-06-07-iqs-central-cpf-mismatch-reg4-value36.log` を比較した。

右手側ログでは、central からの CCC write は `reg 0` にだけ来ている。

~~~text
split_input_events_ccc: reg 0 value 1 subscribed 1
zmk_split_bt_report_input: Dropping input split event reg 3; no subscriber
~~~

一方で、左手 central 側ログでは同じ接続時に以下が出ている。

~~~text
Found an input characteristic
Found input CCC descriptor
Found input CPF descriptor
Input split reg 4 value_handle 36 ccc_handle 37
~~~

この組み合わせは矛盾している。
`value_handle 36 / ccc_handle 37` への subscribe は右手側では `reg 0` の CCC write として観測されている。
しかし central は、その subscribe 対象を `reg 4` として記録している。

原因は、central 側の input characteristic discovery で CCC を見つけた後、CPF descriptor を split service 終端まで広く検索していたこと。
Zephyr の `BT_GATT_DISCOVER_STD_CHAR_DESC` は UUID 指定で範囲内を検索するため、検索範囲が広いと次の input characteristic の CPF を拾える。
その結果、central は `reg 0` の characteristic に subscribe しながら、別 characteristic の CPF を読んで `reg 4` と誤認していた。

対応:

- central の `peripheral_input_slot` に `service_end_handle` を追加した。
- input characteristic ごとの CCC を見つけた後、CPF 検索範囲を `ccc_handle + 1` の1ハンドルだけに制限した。
- CPF を読み終えた後、characteristic discovery の `end_handle` を元の service end に戻すようにした。
- これにより、`value_handle`、`ccc_handle`、`reg` の組が別 characteristic と混ざらないようにした。

この変更後の `./just build Polaris_L_UNIFIED` は成功した。

- artifact: `firmware/zmk-config-GeaconPolaris/Polaris_L_UNIFIED.uf2`
- size: 1012224 bytes
- SHA256: `19a40597ce5411ca659a55ded67e3aceb736a2455c347ef490cb821c761ac238`

次の実機確認では、左手にこの artifact を書き込み、central serial log に以下が出るかを見る。

- `Input split reg 0 ...`
- `Input split reg 3 ...`
- `Input split reg 4 ...`

右手側では、IQS 操作時に以下が消えるかを見る。

~~~text
Dropping input split event reg 3; no subscriber
~~~

## 2026-06-07 追加確認 40: 右手側ではまだ `reg 3` が subscribe されていない

添付ログ `2026-06-07-iqs-reg3-still-not-subscribed-right-only.log` は右手 peripheral 側のみのログ。

確認できたこと:

- IQS profile は settings から復元されている。
- `i2c@40004000` と `iqs9151@56` は初期化されている。
- IQS9151 は IRQ、frame read、input report を生成している。
- central からの CCC write は引き続き `reg 0` のみ。
- IQS の `reg 3` input event は `no subscriber` で drop されている。

重要行:

~~~text
split_input_events_ccc: reg 0 value 1 subscribed 1
iqs9151: report #0 rel code=0x0000 value=-36 sync=0
zmk_split_bt_report_input: Dropping input split event reg 3; no subscriber
~~~

このログだけでは、左手 central に `追加確認 39` の修正済み artifact が書き込まれているか判断できない。
判断に必要なのは左手 central 側ログ。

今回追加した central 側の補強:

- input descriptor 検索が空振りした場合、そこで discovery 全体を止めず、次の characteristic discovery へ復帰する。
- input split characteristic の `value_handle` と `service_end` をログ出力する。
- CCC handle をログ出力する。
- CPF handle、reg、value_handle、ccc_handle の組をログ出力する。

更新後の `./just build Polaris_L_UNIFIED` は成功した。

- artifact: `firmware/zmk-config-GeaconPolaris/Polaris_L_UNIFIED.uf2`
- size: 1013760 bytes
- SHA256: `63f65ab375b9aeec6fbe0018172a6eee5e804b52314ee1fcc98081573ab3e910`

次の実機確認では、左手にこの artifact を書き込み、左手 central log で以下を見る。

~~~text
Input split characteristic value_handle ...
Input split CCC handle ...
Input split reg 0 cpf_handle ... value_handle ... ccc_handle ...
Input split reg 3 cpf_handle ... value_handle ... ccc_handle ...
Input split reg 4 cpf_handle ... value_handle ... ccc_handle ...
~~~

右手側では、`split_input_events_ccc: reg 3 value 1 subscribed 1` が出るかを見る。

## 2026-06-07 追加確認 41: descriptor discovery 挿入で後続 input characteristic を落としている

添付ログ `2026-06-07-iqs-central-only-reg0-before-direct-cpf-read.log` は左手 central 側のログ。

確認できたこと:

- central は右手 split service に接続できている。
- 最初の input characteristic は見つかっている。
- その input characteristic は `value_handle 36 / ccc_handle 37`。
- CPF 読み取り後、`reg 0` として購読している。
- その後、`reg 3` と `reg 4` の input characteristic は発見されていない。
- discovery は `select physical layout` へ進んでいる。

重要行:

~~~text
Found an input characteristic
Input split characteristic value_handle 36 service_end 65535
Input split CCC handle 37 for value_handle 36
Input split reg 0 cpf_handle 41 value_handle 36 ccc_handle 37
Found select physical layout handle
~~~

この結果から、central の input discovery 中に `BT_GATT_DISCOVER_STD_CHAR_DESC` を挟む方式は不安定と判断した。
少なくとも今回のログでは、最初の input characteristic の descriptor discovery 後に、後続の input characteristic discovery が期待通り続いていない。

対応:

- input characteristic discovery 中に CCC/CPF discovery を挟む方式をやめた。
- input characteristic を見つけたら、まず characteristic discovery は継続する。
- CCC handle は `value_handle + 1` として扱う。
- CPF handle は `value_handle + 2` として `bt_gatt_read()` で直接読む。
- CPF read callback で `description` から `reg` を取り出し、その時点で該当 input characteristic を subscribe する。

この前提は右手 peripheral 側の `INPUT_SPLIT_CHARS` マクロと一致している。

~~~c
BT_GATT_CHARACTERISTIC(...)
BT_GATT_CCC(...)
BT_GATT_DESCRIPTOR(BT_UUID_GATT_CPF, ...)
~~~

更新後の `./just build Polaris_L_UNIFIED` は成功した。

- artifact: `firmware/zmk-config-GeaconPolaris/Polaris_L_UNIFIED.uf2`
- size: 1014784 bytes
- SHA256: `8f110c4376a0a574dbd6dfaddd51139e611c6b44f2ac33c4ea820ef23774e9a0`

次の実機確認では、左手 central log に以下が出るかを見る。

~~~text
Input split characteristic value_handle 36 ccc_handle 37 cpf_handle 38
Input split characteristic value_handle 40 ccc_handle 41 cpf_handle 42
Input split characteristic value_handle 44 ccc_handle 45 cpf_handle 46
Input split reg 0 cpf_handle 38 value_handle 36 ccc_handle 37
Input split reg 3 cpf_handle 42 value_handle 40 ccc_handle 41
Input split reg 4 cpf_handle 46 value_handle 44 ccc_handle 45
~~~

右手側では、以下が出るかを見る。

~~~text
split_input_events_ccc: reg 3 value 1 subscribed 1
~~~

## 2026-06-07 追加確認 42: IQS が central まで到達し実動作した

`Polaris_L_UNIFIED` / `Polaris_R_UNIFIED` の IQS profile で、右手 peripheral から central へ IQS 入力が届くことを確認した。

保存ログ:

- `docs/logs/2026-06-07-iqs-success-central-reg3-input-events.log`
- `docs/logs/2026-06-07-iqs-success-right-reg3-subscribed-log-cap.log`

確認できたこと:

- central は input split の `reg 0` / `reg 3` / `reg 4` を direct CPF read で正しく認識した。
- right peripheral は `reg 0` / `reg 3` / `reg 4` の CCC subscribe を受け取った。
- IQS report は `reg 3` の input event として central に到達した。
- central は `Got peripheral event for 3!` を出し、`zmk_hid_mouse_movement_set` まで到達した。
- 実機観測として、IQS profile で入力が動作した。

重要なログ:

~~~text
Input split reg 0 cpf_handle 38 value_handle 36 ccc_handle 37
Input split reg 3 cpf_handle 42 value_handle 40 ccc_handle 41
Input split reg 4 cpf_handle 46 value_handle 44 ccc_handle 45
split_input_events_ccc: reg 3 value 1 subscribed 1
zmk_input_split_report_peripheral_event: Got peripheral event for 3!
zmk_hid_mouse_movement_set: Mouse movement set to 0/-6
~~~

右手側の `iqs9151` ログは `irq #15` / `report #15` 以降で止まって見える。
ただし central 側では `reg 3` input event と mouse movement が確認できており、実機でも動作した。
そのため、この現象はフリーズではなく IQS driver の診断ログ出力上限によるものと判断する。

## 2026-06-07 追加対応 43: IQS の inertia / scroll 処理が体感に出ない問題

IQS profile の入力は central まで届くようになったが、慣性スクロールなどの IQS 向け処理が体感に出ていない。

確認したこと:

- `Polaris_R_UNIFIED` では `CONFIG_INPUT_IQS9151_CURSOR_INERTIA_ENABLE=y` と `CONFIG_INPUT_IQS9151_SCROLL_INERTIA_ENABLE=y` が有効。
- 右手側 IQS driver は inertia 時に `INPUT_REL_X/Y` または `INPUT_REL_WHEEL/HWHEEL` を出す実装になっている。
- 直近の成功ログでは、右手側 IQS は `INPUT_REL_X/Y` の report だけを出していた。
- central 側では `reg 3` の input event が `trackpad_listener` に入り、既存の右手 trackpad 用 processor で処理されていた。
- central 側の `trackpad_listener` は scroll に `zip_scroll_scaler 1 50` を使っていたため、IQS inertia が小さい wheel 値を出しても 0 に潰れやすい。
- `CONFIG_ZMK_POINTING_SMOOTH_SCROLLING` の `apply_resolution_scaling()` は `scaled` を計算していたが、実際には `evt->value = val` になっており、計算結果を使っていなかった。

対応:

- central 側 `trackpad_listener` に `zip_dynamic_xy_scaler` / `zip_dynamic_scroll_scaler` を追加した。
- central 側 `trackpad_listener` の scroll 固定スケールを `1/50` から `1/20` へ、lowspeed を `1/70` から `1/35` へ緩めた。
- ZMK の smooth scroll scaling を `evt->value = scaled` に修正した。
- IQS driver に `cursor inertia start` / `scroll inertia start` / `... step` の診断ログを追加した。

ビルド:

- `./just build Polaris_L_UNIFIED` 成功。
- `./just build Polaris_R_UNIFIED` 成功。

次の実機確認:

- 1本指を素早く動かして離したときに、右手ログへ `cursor inertia start` と `cursor inertia step` が出るか確認する。
- 2本指スクロールをして離したときに、右手ログへ `scroll inertia start` と `scroll inertia step` が出るか確認する。
- central 側で `INPUT_REL_WHEEL` / `INPUT_REL_HWHEEL` が mouse scroll として出るか確認する。

## 残っている確認項目

Module profile の残検証:

- 左手側 TB profile で spi2 と PMW3610 が初期化されること。TB profile は実機で動作確認済みだが、左右個別の詳細ログは未取得。
- 左手側 JOY profile で ADC と encoder が動くこと。実機で動作確認済み。詳細ログは未取得。
- 左手側 ENC profile で encoder が動くこと。EC11 polling trigger + system work forwarding + `steps=<96>` 版で実機動作確認済み。
- 右手側 TPD profile で i2c0 と glidepoint_R が初期化されること。
- 右手側 IQS profile で central が `reg 3` を subscribe し、入力が central に届くこと。実機動作確認済み。
- IQS profile 時に central の reason `0x08` 切断が再発するか確認すること。現在の成功ログでは IQS 入力は central まで届いている。
- IQS profile の長時間操作で BLE / ログ詰まりがないか確認すること。
- 左右で別 profile を保存し、cold boot 後もそれぞれ復元されること。

別件として扱うもの:

- qspi_nor の JEDEC ID mismatch。
- 古い input_proc/scroll settings key の error(-2)。
- battery level undetermined / zero。
