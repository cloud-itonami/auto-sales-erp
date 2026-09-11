# auto-sales-erp — data-center-ops の BPMN wire contract bundle

**この repo の名前は中身を言っていない。** `auto-sales-erp` という名前だが、
入っているのは自動車販売 ERP ではなく、**data-center-ops actor の
BPMN 2.0 プロセス契約 2 本**である。実行するコードは 1 行も無い。

名前がこうなっているのは、抽出元のパスがそうだったからで（`migration.edn`:
`etzhayyim/root` の `60-apps/etzhayyim-project-auto-sales-erp`、2026-07-19 抽出）、
このワークスペースの規則では**移転や実態の変化で repo を改名しない**
（名前は discovery alias であって identity ではない。superproject CLAUDE.md /
ADR-2608040100）。だから名前は据え置き、代わりにここで名乗る。

## 中身

| ファイル | プロセス | 構造 |
|---|---|---|
| `wire/data-center-ops-operations.bpmn` | `Process_DataCenterOps_Operations`「Data Center Operations Management」 | 9 node / 9 flow / service-task 6 / exclusive-gateway 1 |
| `wire/data-center-ops-dependency-reverse-topo.bpmn` | `Process_DataCenterOps_DependencyReverseTopo`「Data Center Dependency Reverse Topology」 | 7 node / 6 flow / service-task 5 / gateway 0 |

targetNamespace は `https://etzhayyim.com/bpmn/data-center-ops` と
その `/dependency`。両方とも `isExecutable="true"` で、BPMN DI（描画座標）を持つ。

**operations** — Schedule Tick → 設備テレメトリ収集 → 逆依存トポロジ解決 →
ラック容量評価 → SLA 適合チェック → `Gateway_Risk`（`default="Flow_6"`）→
risk あり: インシデント起票・エスカレーション → ダッシュボード更新 /
risk なし: ダッシュボード更新 → Cycle Complete。

**dependency-reverse-topo** — Request Dependency View → baseline seed →
node 列挙 → edge 列挙 → global トポロジ収集 → 逆トポロジカル順の解決 →
Dependency Graph Ready。分岐は無い直列。

## 実装はここではなく `cloud-itonami/data-center-ops` にある

service task の名前に Lexicon NSID が埋まっており、その NSID を宣言している
のは**兄弟 repo の `orgs/cloud-itonami/data-center-ops`**（actor scaffold、
`did:web:data-center-ops.etzhayyim.com`）。宣言の正本はあちらの
`actor-manifest.jsonld`（NSID 22 件。下表の 5 件は全部そこに実在する）。
この repo は契約だけを持ち、束縛先を持たない。

**束縛は完全ではない。** dependency 側は 5 task 全部が NSID を持つが、
operations 側は 6 task 中 1 つ（`Task_ResolveReverseTopo` →
`…dependency.getReverseTopo`）だけで、
`CollectTelemetry` / `EvaluateCapacity` / `CheckSla` / `CreateIncident` /
`UpdateDashboard` は**どの XRPC にも束縛されていない**。BPMN としては妥当
（service task に実装束縛は必須ではない）。

**「NSID が無いから未実装」と読まないこと。** actor 側の manifest には
`infrastructure.listRacks` / `infrastructure.getSlaSummary` /
`dataCenterOps.incident` など、この 5 つの相手になりそうな NSID が実在する。
つまり欠けているのは実装ではなく**束縛の記述**である。どれがどれに対応するかは
どこにも書かれていないので、**対応を推測で埋めない** —— 埋めるときは actor 側で
実在を確かめてから task 名に書く（手順は quickstart 末尾）。

| BPMN task | Lexicon NSID |
|---|---|
| `Task_SeedBaseline` | `com.etzhayyim.apps.dataCenterOps.dependency.seedBaseline` |
| `Task_ListNodes` | `com.etzhayyim.apps.dataCenterOps.dependency.listNodes` |
| `Task_ListEdges` | `com.etzhayyim.apps.dataCenterOps.dependency.listEdges` |
| `Task_CollectGlobal` | `com.etzhayyim.apps.dataCenterOps.dependency.collectGlobal` |
| `Task_ReverseTopo` / `Task_ResolveReverseTopo` | `com.etzhayyim.apps.dataCenterOps.dependency.getReverseTopo` |

⚠ **兄弟 repo 側の散文は抽出前のパスを指したままなので、そのまま辿らないこと。**
`data-center-ops/README.md` は (a) この 2 本を
`etzhayyim-root/60-apps/etzhayyim-project-auto-sales-erp/bpmn/*.bpmn` で指し
（現在地はこの repo の `wire/`）、(b) lexicon を
`00-contracts/lexicons/com/etzhayyim/apps/dataCenterOps/` と書いているが
**その path はあちらの repo に存在しない**（実測: あちらは 6 ファイルのみ）。
機械可読な `actor-manifest.jsonld` の方は実在し、NSID もそこで引ける。

## 検査する

コードを持たない contract bundle なので、「壊れていない」は
**仕様どおり読めて構造検査に通ること**しか意味しない。それを 1 コマンドで確かめる:

```bash
ROOT=~/github/com-junkawasaki                       # west superproject root
kbb --backend sci --classpath "$ROOT/orgs/kotoba-lang/org-omg-bpmn/src" scripts/validate-contracts.cljk
```

検査規則はこの repo が持たず、`kotoba-lang/org-omg-bpmn` の `bpmn.validate`
に委ねている（dangling flow・未知 node 型・到達不能 node・決定不能 gateway）。
手順と期待出力、壊したときに何が出るかは **[`docs/operator-quickstart.md`](docs/operator-quickstart.md)**。

## この repo が持たないもの

- **実行系** — engine も runtime も無い。BPMN を実際に回すのは actor 側。
- **deps.edn** — 依存を持たない（検査は上流ライブラリを classpath で借りる）。
- **テスト** — 上の検査コマンドが実質的にその役目を負う。
- **ドメインデータ** — baseline に何が入るか（land / facility / permit / ISCO
  workforce / APQC / power / rack / server / license）は兄弟 repo の README の
  散文に列挙がある。ただしそれが指す RisingWave migration
  （`30-graph/graph-schema/migrations/…_data_center_ops_dependency_graph.ts`）も
  **あちらの repo には無い** —— 上の ⚠ と同じ、抽出前を指したままの参照なので、
  実体を当てにするなら先に所在を確かめること。

## 出所

`migration.edn` に抽出元を pin してある（`etzhayyim/root`
`60-apps/etzhayyim-project-auto-sales-erp` @ `739557df`、2026-07-19）。
`README.edn` は west/RAD が読む機械可読な面で、この README.md と同じことを
EDN で言っている。
