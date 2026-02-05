# JSONLDインポート機能改修 詳細設計書

## 1. 文字列置換関数仕様

### 1.1 JsonLdMapper.apply_import_replace_rules

#### 実装箇所

modules/weko-search-ui/weko_search_ui/mapper.py

#### 目的・概要

受け取った`metadata`（dict）に対し、設定ファイル(instance.cfg)で定義されたインポート時の置換ルールを適用し、文字列置換を行った上で辞書型としてreturnする。
ワーニングが発生した場合は`info["warnings"]`に追加する。

#### 引数

- metadata (dict): インポート対象のメタデータ
- info (dict): インポート処理中の補助情報

#### 戻り値

- tuple
    - metadata (dict): 置換処理後のメタデータ
    - info (dict): ワーニング情報を含む補助情報

#### 処理仕様

1. ワーニングリスト（`warning_list`）を空リストで初期化する。
2. `self.mapping_id`から`mapping_id`を取得する。
3. `current_app.config`から[`WEKO_SEARCH_UI_IMPORT_REPLACE_RULES`](#31-weko_search_ui_import_replace_rules)(`rules`)と[`WEKO_SEARCH_UI_IMPORT_REPLACE_RULE_MAP`](#32-weko_search_ui_import_replace_rule_map)(`rule_map`)を取得する。
4. `rule_map`から`mapping_id`をキーにして、適用するルールキーリスト（`rule_keys`）を取得する。
5. `rules`、`rule_map`、`rule_keys`の型チェックを行い、型に誤りがある場合は`ValueError`を発生させる。
6. `rule_keys`をループし、各`rule_id`ごとに以下を実施する。
    - `rules`に`rule_id`がなければワーニングリストに追加し、次のルールへ進む。
    - `rule`から置換前文字列（`from_str`）、置換後文字列（`to_str`）、置換対象パスリスト（`jsonld_path_list`）を取得する。
    - `from_str`が「str型」で「空文字でない」、`to_str`が「str型」、`jsonld_path_list`が「list型」であることを確認し、不正なら`warning_list`に追加して次の処理へ進む。
    - 正規表現フラグ（`is_regex`）はbool型でなければFalseとする。
    - `jsonld_path_list`内の各`path_key`と、list変換した`metadata`の各キーをループで順番に確認し、インデックスを除去した`meta_key_no_index`が`path_key`と一致する場合に置換処理を行う。
        - `is_regex`がTrueの場合は、`re.sub`を用いて`from_str`→`to_str`の正規表現置換を行う。       
        このとき、`to_str`に正規表現(`r"..."`)などを誤って指定され意図しない置換になることを防ぐため、常に`to_str`の値をそのまま返すlambda関数を`re.sub`の置換関数として使用する。   
        - `is_regex`がFalseの場合は`replace`で`from_str`→`to_str`の単純置換を行う。
7. `warning_list`に値があれば`ValueError`を発生させ、[例外処理](#例外処理)に進む。
8. `warning_list`に値がなければ、置換後の`metadata`と`info`をreturnする。

#### 例外処理

- 例外発生時は`info["warnings"]`にワーニングメッセージを追加し、`metadata`と`info`をreturnする。
    - 上記処理仕様5.で例外処理に入った場合は**置換前**の`metadata`、    
    処理仕様7.で例外処理に入った場合は**置換後**の`metadata`となる。
- `ValueError`でリスト型のメッセージの場合はリストをループさせ、リスト内の各ワーニングを`info["warnings"]`に追加する。
- それ以外の例外は内容を`info["warnings"]`に追加する。

#### 例外メッセージ 言語設定

- 英語の場合は`Replacement failed.: {e}`、日本語の場合は`置換処理に失敗しました。: {e}`と表示する。
- `{e}`には 以下の表に記載のメッセージが入る。

    | 内容                                                             | 英語メッセージ                                              | 日本語メッセージ                                | 備考                    |
    | :--------------------------------------------------------------- | :---------------------------------------------------------- | :---------------------------------------------- | :---------------------- |
    | 置換ルール定義、置換ルールマッピング、<br>ルールキーリストの型が不正 | The type of the jsonld mapping replacement rule is invalid. | jsonldマッピングの置換ルールの型が不正です。    |                         |
    | 置換ルールIDが取得できない    | Required replacement rule: '{rule_id}' is missing.          | 必要な置換ルール：'{rule_id}'が見つかりません。 | {rule id}: 置換ルールID |
    | `from`、`to`、`jsonld_path`の設定が不正    | Replacement rule: '{rule_id}' is invalid.                   | 置換ルール： '{rule_id}'の設定が不正です。      | {rule id}: 置換ルールID |
    | それ以外のエラー発生時                                           | {起きたエラーのメッセージ}                                  | {起きたエラーのメッセージ}                      |                         |

#### 備考

- ワーニングがあっても、インポート処理上置換が必要なかった場合はインポート処理は正常に実行される。

---

## 2. 置換関数呼び出し箇所・設定

### 2.1 JsonLdMapper._map_to_item

#### 実装箇所
modules/weko-search-ui/weko_search_ui/mapper.py/_map_to_item

#### 呼び出し方法

`_map_to_item`関数内でアイテムのメタデータ本体(`metadata`)と、システム情報(`system_info`)を定義している。   
`system_info`定義後に、`metadata`と`system_info`を使用して[置換処理](#11-jsonldmapperapply_import_replace_rules)を呼び出しを行う。

```python
# 既存コード
mapped_metadata = {}
system_info = {
    ... # 省略
}

# system_info定義後に置換処理の呼び出し
metadata, system_info = self.apply_import_replace_rules(metadata, system_info)
```

### 2.2 mapping_idの設定

### 実装箇所
modules/weko-search-ui/weko_search_ui/utils.py/check_jsonld_import_items

### 目的
[apply_import_replace_rules](#11-jsonldmapperapply_import_replace_rules)で利用する置換ルールを特定するため、mapping_idをインスタンスにセットする。

### 設定方法
```python
mapper.mapping_id = mapping_id
item_metadatas, _ = mapper.to_item_metadata(json_ld) # 既存コード
```

---

## 3. 置換ルール定義

### 3.1 WEKO_SEARCH_UI_IMPORT_REPLACE_RULES

#### 実装箇所
scripts/instance.cfg    
modules/weko-search-ui/weko_search_ui/config.py

#### 型定義
辞書型(dict)

#### 概要
インポート時に適用する文字列置換ルールの定義を保持する辞書。各ルールはルールキーによって識別される。

ルールキーは以下のような構成で定義する。
```Properties
ルール名: {
    "from": ...,
    "to": ...,
    "is_regex": ...,
    "jsonld_path": [...]
}
```

- ルール名      
    ルール定義のキー名。    
    同じキー名を複数用いた場合、同一キー名の最後のルール定義のみ有効になる。

- 各要素
    |   要素名   |    必須  |   型   |    空文字<br>指定    |複数<br>指定 |概要  |   備考 |
    |:---------:|:---------:|:---------:|:---------:|:---------:|:---------|:-----|
    |   from |  〇  |   str  |   ×    | 〇 |  置換**前**の文字列を指定する。| 正規表現を使用する場合、r"(...)\|(...)"とすることでor検索を行う。  |
    |   to   |    〇    | str    | 〇 |  ×   |    置換**後**の文字列を指定する。|   ・ 空文字を指定した場合は置換前の文字列が削除される。<br>・指定した文字列にそのまま置換されるため、**正規表現ではなく**、単一の文字列を指定する。    |
    |   is_regex |  ×   |    bool  |   ×    | ×  |   Trueの場合は正規表現が一致した文字列を置換、Falseの場合は完全一致した文字列を単純置換する。   |  指定しなかった場合はFalseとなる。   |
    |   jsonld_path  |   ×    | list   |    × |  〇  |   `jsonld_mappings`テーブルの`mapping`カラムで定義されたプロパティ名(JSON-LDマッピングの右側の値)を指定する。  |   ・空リストの場合は該当のルール定義の置換処理をスキップする。<br>・`"データ作成者.作成者姓名.姓名": "creator.name.value"`を置換対象としたい場合、<br>右側の`"creator.name.value"`を指定する。|    

#### 実装例 

is_regexのTrue/Falseでfromの書き方が変わるため、例を記載する。      
実装時点、instance.cfgには`pipe_full_width(is_regex:False)`のみを実装し、config.pyは空dictの`WEKO_SEARCH_UI_IMPORT_REPLACE_RULES`を定義する。

```Properties
WEKO_SEARCH_UI_IMPORT_REPLACE_RULES = {
    # is_regex=False:　完全一致した置換元文字列(from)を置換後文字列(to)に置換する定義方法。こちらを実装時点のデフォルトとする。
    "pipe_full_width": {
        "from": "|",
        "to": "｜",
        "is_regex": False,
        "jsonld_path": [
            "ams:industrialUse.value", 
            "ams:anonymousProcessing.value"
        ]
    },
    # is_regex=True: 正規表現にマッチした置換元文字列(from)を置換後文字列(to)に置換する定義方法。
    "space_full_width": {
        "from": r"(\u3000)|(　)",
        "to": " ", # toには正規表現は使えない
        "is_regex": True,
        "jsonld_path": [
            "contributor.name.value"
        ]
    }
}
```

#### 注意点

1. 文字列置換ルールに同一のルール名を複数回用いた場合   
    同一ルール名のうち、最後のルール定義が有効になる。最後以外のルール定義は使用されない。

    例： 同一ルール名を定義した場合
    ```Properties
    WEKO_SEARCH_UI_IMPORT_REPLACE_RULES = {
        # 無効
        "rule_name": {             
            "from": "abc",
            "to": "あ",
            "is_regex": False,
            "jsonld_path": [
                "abc.value"
            ]
        },
        # 無効
        "rule_name": {             
            "from": "abc",
            "to": "い",
            "is_regex": False,
            "jsonld_path": [
                "abc.value"
            ]
        },
        # 有効
        "rule_name": {              
            "from": "abc",
            "to": "う",
            "is_regex": False,
            "jsonld_path": [
                "abc.value"
            ]
        }
    }
    ```
2. `to`に正規表現を用いた場合   
    toに指定した文字列は**そのままの文字列**として置換されるため、正規表現を用いると意図した置換にはならない。
    ```Properties
    "rule_name": {             
        "from": r"\|",
        "to": r"\uFF5C",　# 全角パイプ相当の正規表現
        "is_regex": True,
        "jsonld_path": [
            "abc.value"
        ]
    }
    ```
    上記のように指定すると、fromに指定した`|`(半角パイプ)が`\\\\uFF5C`(バックスラッシュ+uFF5C)に置換される。  

    以下のようにUnicodeエスケープであれば指定可能。
    ```Properties
    "rule_name": {             
        "from": r"\|",
        "to": "\uFF5C",　# 全角パイプ相当のUnicodeエスケープ
        "is_regex": True,
        "jsonld_path": [
            "abc.value"
        ]
    }
    ```
    上記の書き方であれば、fromに指定した`|`(半角パイプ)が`｜`(全角パイプ)に置換される。

### 3.2 WEKO_SEARCH_UI_IMPORT_REPLACE_RULE_MAP

#### 実装箇所
scripts/instance.cfg    
modules/weko-search-ui/weko_search_ui/config.py

#### 型定義
辞書型(dict)

#### 概要
`jsonld_mappings`テーブルの`id`(mapping_id)をキーとし、適用すべき置換ルール（`WEKO_SEARCH_UI_IMPORT_REPLACE_RULES`のキー）の値リストを持つ辞書。   
各mapping_idごとにどの置換ルールを適用するかを制御する。

#### 実装例
実装時点、instance.cfgには以下のように実装する。    
config.pyは空dictの`WEKO_SEARCH_UI_IMPORT_REPLACE_RULE_MAP`を定義する。
```Properties
WEKO_SEARCH_UI_IMPORT_REPLACE_RULE_MAP = {
    "32001": [    # jsonld_mappingsテーブルのid(mapping_id)
        "pipe_full_width"
    ]
}
```

---

## 4. researchmap連携機能仕様

### 4.1 wkコンテキストの追加

#### 修正箇所
JSON-LDのカスタム語彙

#### 目的・概要
- wk:researchmapLinkageプロパティを追加し、この値に応じて連携処理を制御する。

#### 処理仕様
- ro-crate-metadata.jsonの`@id: "./"`のエンティティに`wk:researchmapLinkage`プロパティを追加する。

    ```json
    "@graph": [
        {
        "@id": "./",
        ...,
        "wk:researchmapLinkage": false
        }
        ...
    ]
    ```

    `true`の場合はresearchmap連携のキュー追加を行い、`false`の場合は何もしない。    
    なお必須項目ではないため、未定義の場合はresearchmap連携のキュー追加は行わない。

---

### 4.2 researchmap連携 キュー追加処理

- ### 4.2.1 Swrod API直接登録、RO-Crateインポート（画面）対応

    #### 実装箇所

    modules/weko-search-ui/weko_search_ui/mapper.py/_deconstruct_json_ld   

    #### 目的・概要

    - JSONLDインポート時にresearchmap連携の有無を判定し、連携処理のキュー追加を行えるようにする。
    - tsvインポートについては既にresearchmap連携のキューを追加する処理を実装済み。       
    JSONLDインポート時にも同一のキュー追加処理が行えるようにする。

    #### 処理仕様

    - 既存のsystem_infoの追加・更新処理に以下の処理を追加する。
        - [4.1 wkコンテキストの追加](#41-wkコンテキストの追加)で設定した`wk:researchmapLinkage`の値を`system_info["researchmap_linkage"]`に代入する。    
        - wk:researchmapLinkageが未定義等で取得出来なかった場合は、`False`を`system_info["researchmap_linkage"]`に代入する。
    - utils.pyのimport_items_to_systemで行われるインポート処理内でresearchmap連携処理のキュー追加を行う。(既存処理)
        ```python
        # 以下は既存コード
        if item.get("researchmap_linkage"):
            pid = PersistentIdentifier.query.filter_by(
                pid_type="recid", pid_value=item["id"]
            ).first()
            cris_researchmap_linkage_request.send(pid.object_uuid)
        ```

- ### 4.2.2 Swrod APIワークフロー登録対応

    #### 実装箇所

    - modules/weko-swordserver/weko_swordserver/views.py/post_service_document/process_item
    - modules/weko-swordserver/weko_swordserver/views.py/put_object
    - modules/weko-workflow/weko_workflow/headless/activity.py/_input_metadata

    #### 目的・概要

    - APIワークフロー登録を用いたアイテムインポート時にresearchmap連携の有無を判定し、連携処理のキュー追加を行う機能を実装する。
    JSONLD、tsvインポートの双方で使用出来るようにする。

    #### 処理仕様

    - `process_item`、`put_object`で`register_type`が`Workflow`の時、`item`から[4.2.1 Swrod API直接登録、RO-Crateインポート（画面）対応](#421-swrod-api直接登録ro-crateインポート画面対応)で登録した`researchmap_linkage`を取得し、`item["metadata"]["researchmap"]`に代入する。
    - _`input_metadata`で`metadata.pop("researchmap", False)`を行い、`researchmap_linkage`を取得し、`metadata`内にある`researchmap`はキーごと削除する。
    - `_input_metadata`の`data`を以下のように変更し、`researchmap`を渡せるようにする。
        ```python
        data = {
                    "metainfo": metadata,
                    "files": self.files_info,
                    "cris_linkage": {　                     # 追加
                        "researchmap": researchmap_linkage　# 追加
                    },　                                    # 追加
                    "endpoint": {
                        "initialization": f"/api/deposits/redirect/{pid.pid_value}",
                    }
                }
        ```
    - weko_workflow/views.pyの`next_action`で行われる承認処理内でresearchmap連携処理のキュー追加を行う。(既存処理)
        ```python
        # 以下は既存コード
        temp_data = work_activity.get_activity_metadata(activity_id=activity_id)
        if temp_data:
            if json.loads(temp_data).get('cris_linkage',{}).get('researchmap' , False):
                cris_researchmap_linkage_request.send(new_item_id)
        ```

---

### 4.3 Ro-Crateエクスポートファイルのresearchmap連携対応

#### 実装箇所

modules/weko-search-ui/weko_search_ui/mapper.py/to_rocrate_metadata

#### 目的・概要

- アイテムエクスポート(Ro-Crate)を実行してエクスポートされるアイテムに`wk:researchmapLinkage`が出力されるようにする。

#### 処理仕様

- `rocrate.root_dataset["wk:researchmapLinkage"]`に値はFalseを設定する。
- `rocrate`をreturnする。（既存処理）