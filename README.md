# 選考課題
特定日時点の予定フリー在庫数

# DB設計
## テーブル追加
①日別の在庫数集計テーブル
→inventory_daily_movements

```
inventory_daily_movements {
    bigint id PK
    bigint company_id FK # アカウントID
    bigint inventory_id FK # 在庫ID
    date scheduled_date #入出庫予定日
    decimal purchase_quantity #入庫予定数量の合計
    decimal delivery_quantity # 出庫予定数量の合計
    datetime created_at # 作成日時
    datetime updated_at # 更新日時
}
```

②入出庫予定日(scheduled_date)が変更された場合の再集計依頼用テーブル
```
inventory_movement_recalculation_requests {
    bigint id PK
    bigint company_id FK # アカウントID
    bigint inventory_id FK # 在庫ID
    date scheduled_date #入出庫予定日
    datetime created_at # 作成日時
    datetime updated_at # 更新日時
}
```
## カラム追加
なし

## index追加
①inventory_daily_movementsテーブルにunique_indexを追加
unique_index[company_id, inventory_id, scheduled_date]
同じ会社・在庫・予定日の集計データが重複しないようにする 

②inventory_movement_recalculation_requestsテーブル
→unique_index[company_id, inventory_id, scheduled_date]
同じ会社・在庫・予定日の集計データが重複しないようにする 

③delivery_itemテーブル
→index[company_id, inventory_id, status, scheduled_date]
集計の時の検索用(日付とステータスでwhereする用)


④purchase_itemテーブル
→index[company_id, inventory_id, status, scheduled_date]
集計の時の検索用(日付とステータスでwhereする用)

# インフラ設計
変更なし

# 実装の方針
## 特定日時点の予定フリー在庫数の算出方法
集計テーブルを利用して計算を行っておく(在庫に対して入出庫予定数を計算しておく)

備考：
考えた選択肢
①都度、入出庫明細から計算する
②集計テーブルを利用して計算を行っておく(在庫に対して入出庫予定数を計算しておく)
→②を選択
┗①にすると都度計算が走り負荷が大きいため、集計テーブルを利用しての計算を選択(入出庫の増減数がすでに計算されていれば在庫数から指定日までの入出庫予定数を加減する計算だけで済むため。)

## 集計テーブルについて
非同期で更新する

備考：
考えた選択肢
①同期的に更新する(トランザクション内)
②非同期で更新する
→②を選択
┗3分の遅延が許されているので、予定フリー在庫については非同期で処理していく、将来的に更新数が増えた場合に非同期の方が対応しやすいと考えたため

①同期の場合
メリット：集計テーブルの反映までに遅延がない
デメリット：1件1件処理が走るので、負担がある、トランザクションが長くなるとロック解除の時間がかかり、同じ在庫を更新する処理が待たされやすくなる、同じ在庫の更新が短時間で複数あってもまとめてではなく1件ずつ処理される
②非同期の場合
メリット：変更をまとめられる、変更したときの処理の負荷が非同期部分にも分散できる
デメリット：反映まで遅延がある、排他制御などの実装や運用の手間が増える

### 集計テーブルの非同期更新方法
ある程度まとめて更新を行う

備考：
考えた選択肢
①入出庫記録の更新ごとにジョブを発行する
②ある程度まとめて更新を行う
→②を選択
①については処理がわかりやすい一方、同じ在庫で複数処理があってもすべてが処理されるため、できればまとめて最新の情報で更新する方が効率が良いと考えたため
入出庫データが作成・更新されるリクエストは20/分なのでまとめても処理が重すぎることはないと想定


・1分ごとに集計するバッチを作成(ECSのScheduled Taskなどで実行)
・バッチはMySQL の GET_LOCK のようなイメージで排他制御を行う
→redisを使うことがあれば任意のlock_keyを設定して同じようなことができる想定
