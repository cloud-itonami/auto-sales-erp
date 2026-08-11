# Operator quickstart

この repo は **BPMN 2.0 の契約 2 本だけ**を持つ（実行系は無い。何なのかは
[`../README.md`](../README.md)）。だから operator の仕事は 1 つしかない ——
**契約が仕様どおり読めて構造検査に通ることを確かめる**こと。

所要 1 分。以下は 2026-08-12 に実際に踏んだ手順と、その場で出た出力である。

## 0. 前提

| 要るもの | 確かめ方 |
|---|---|
| `nbb` | `nbb --version` → 実測 `nbb v1.4.210`（このワークスペースの script host。`bb` は使わない） |
| `kotoba-lang/org-omg-bpmn` の checkout | 下の手順 1 |

検査規則はこの repo に無い。上流ライブラリ `org-omg-bpmn` の `bpmn.validate`
から借りる。west 管理なので、手元に無ければ取ってくる。

## 1. 上流ライブラリを確保する

```bash
ROOT=~/github/com-junkawasaki                       # west superproject root
ls "$ROOT/orgs/kotoba-lang/org-omg-bpmn/src/bpmn/validate.cljc"
```

出れば済み。`No such file` なら superproject 側で取る（**引数なしの
`west update` を打たないこと** —— 4,100 project 全部を歩く）:

```bash
cd "$ROOT" && west update --fetch smart org-omg-bpmn
```

## 2. 契約を検査する

```bash
cd <この repo>
nbb --classpath "$ROOT/orgs/kotoba-lang/org-omg-bpmn/src" scripts/validate-contracts.cljs
```

実際の出力:

```
wire/data-center-ops-dependency-reverse-topo.bpmn
  process Process_DataCenterOps_DependencyReverseTopo  "Data Center Dependency Reverse Topology"
  nodes=7 flows=6 start=1 end=1 service-task=5 gateway=0
  OK  errors=0 problems=0
wire/data-center-ops-operations.bpmn
  process Process_DataCenterOps_Operations  "Data Center Operations Management"
  nodes=9 flows=9 start=1 end=1 service-task=6 gateway=1
  OK  errors=0 problems=0

2 contract(s), 0 error(s)
```

**exit code が判定**（`echo $?` → `0`）。error が 1 件でもあれば `1` で落ちる。
`warn` は落とさない（構造上は妥当だが読み手に伝えたい事柄 —— 到達不能 node、
条件も default も無い gateway など）。

## 3. その検査が本当に落ちることを確かめる

**これを飛ばさないこと。** 「常に緑」の検査は、壊れていないことの証拠ではなく
何も見ていないことの証拠でありうる。壊したコピーで落ちるところまで見て、
初めて手順 2 の `OK` に意味が出る。

```bash
rm -rf /tmp/bpmn-falsify && mkdir -p /tmp/bpmn-falsify
cp -r wire scripts /tmp/bpmn-falsify/
cd /tmp/bpmn-falsify
sed -i '' 's|targetRef="Task_EvaluateCapacity"|targetRef="Task_DoesNotExist"|' \
  wire/data-center-ops-operations.bpmn      # flow の行き先を存在しない node に向ける
nbb --classpath "$ROOT/orgs/kotoba-lang/org-omg-bpmn/src" scripts/validate-contracts.cljs
echo "exit=$?"
```

実際の出力（後半のみ）:

```
wire/data-center-ops-operations.bpmn
  process Process_DataCenterOps_Operations  "Data Center Operations Management"
  nodes=9 flows=9 start=1 end=1 service-task=6 gateway=1
  error [:flow/dangling-target] sequenceFlow Flow_2b targetRef Task_DoesNotExist is not a node
  warn [:node/no-incoming] node Task_EvaluateCapacity has no incoming flow
  FAIL  errors=1 problems=2

2 contract(s), 1 error(s)
exit=1
```

壊した箇所（`Flow_2b`）を名指しし、その巻き添えで到達不能になった
`Task_EvaluateCapacity` も報告する。片付けは `rm -rf /tmp/bpmn-falsify`。

**床も張ってある。** `wire/` から `.bpmn` が 1 本も見えなくなった場合
（パス変更や絞り込みの壊れ）、「0 件検査して 0 error だから合格」にならず
落ちる:

```bash
mkdir -p /tmp/bpmn-empty/wire && cp -r scripts /tmp/bpmn-empty/ && cd /tmp/bpmn-empty
nbb --classpath "$ROOT/orgs/kotoba-lang/org-omg-bpmn/src" scripts/validate-contracts.cljs
# → wire/*.bpmn が 0 件。contract bundle が空になっている   /  exit=1
```

## 4. 契約を EDN として読む（必要なら）

`org-omg-bpmn` は BPMN を素の Clojure データとして扱うので、XML を目で追わずに
トポロジを引ける。分岐先を確かめる例:

```bash
nbb --classpath "$ROOT/orgs/kotoba-lang/org-omg-bpmn/src" -e '
(require (quote [bpmn.xml :as x]) (quote [bpmn.model :as m]) (quote ["node:fs" :as fs]))
(let [mo (x/parse-str (fs/readFileSync "wire/data-center-ops-operations.bpmn" "utf8"))]
  (println (pr-str (m/successors mo "Gateway_Risk"))))'
```

```
("Task_CreateIncident" "Task_UpdateDashboard")
```

他に使えるもの: `m/nodes` / `m/flows` / `m/start-events` / `m/end-events` /
`m/nodes-of-type` / `m/outgoing` / `m/incoming` / `m/predecessors`。
`outgoing`/`incoming` は flow id 順で決定的なので、出力を diff にかけてよい。

## 契約を変更するとき

1. `wire/*.bpmn` を編集する。
2. 手順 2 を通す（**exit 0 を確認する。目視で済ませない**）。
3. service task に XRPC を束縛する/外すなら、兄弟 repo
   `cloud-itonami/data-center-ops` と食い違わせない —— NSID の正本は
   あちらの `actor-manifest.jsonld`（あちらの README が指す
   `00-contracts/lexicons/…` は**実在しないパス**なので使わない）。
4. 現在 operations 側は 6 task 中 1 つしか束縛が無い。埋めるなら、まず
   あちらに NSID が実在することを確かめてから task 名に書く（逆順にすると
   契約が実在しない XRPC を指す）。
