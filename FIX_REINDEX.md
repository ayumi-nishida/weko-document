# Reindex改善 詳細設計書

## count誤差修正


- 実装箇所

    modules/invenio-indexer/invenio_indexer/api.pyのclass RecordIndexer
- 概要

    例外エラー時、出力されるcountの誤差をなくすための修正

- 修正方針

    - process_bulk_queue()をリファクタリングした新たな関数を作成することにより、count誤差が起きないようにする。<br>
    そのため、新たな関数の実装はprocess_bulk_queue()を元にして進める。
    - 例外エラーが発生した場合、リトライせずにエラーとする。
    - エラー発生時にbulkを最後まで実行する設定の場合でも、ログとしてアイテムごとの結果やエラー内容を出力する


- 修正内容

    ### 1. process_bulk_queue_reindex(self, es_bulk_kwargs=None,with_deleted=False)

    - 引数

        - es_bulk_kwargs：bulk APIに渡すキーワード
        - with_deleted：削除済みレコードもインデックス処理の対象とするか

    - 処理

        - 未実施件数を正確に把握するため、関数の頭でmessages_countを宣言し、messagesを取得したタイミングでlenを取得する。
        - 例外エラー（BulkIndexError）発生時
            1. 成功、失敗、未実施数を計算する。<br>BulkIndexErrorはerrorsにエラー情報が集約されるため、errorsからfailを取得する。
            2. エラーが起きたID,およびエラー原因を出力する。
            3. 再bulk()しないため、error_ids = [] 以降の処理を削除する。
        - 例外エラー（BulkIndexError以外）発生時
            BulkIndexError以外の例外エラー発生によりbulkが中断された場合、成功/失敗数を取得出来ない。
            bulk同等、streaming_bulkのラップ関数を作成し、bulkの代わりに呼び出す。
            例外発生時、独自例外クラスで処理を行う。（次項で詳細説明）
            1. 成功、失敗、未実施数を計算する。<br>
            2. エラーが起きたID、およびエラー原因を出力する。
            3. 再bulk()しないため、error_ids = [] 以降の処理は不要。
        - 未実施項目がある場合はcount出力にunprocessedを追加する。<br>
        未実施件数がない場合はsuccess、failのみ出力する。

    - 戻り値

        - count：成功、失敗、未実施件数（タプル）


    ### 2. reindex_bulk(self, client, actions, stats_only=False, *args, **kwargs)

    - 引数
        - client
        - actions
        - stats_only=False
        - *args
        - **kwargs


    -　概要
    
    bulk()中に例外エラーが発生した場合、エラーが発生する前までに成功したデータの登録は完了している。<br>
    しかしbulk()を完走しなければ成功数/失敗数の取得は出来ない。<br>
    bulk()の代わりに以下で作成するstreaming_bulk()のラップ関数を呼び出すようにする。<br>
    例外エラークラスを作成し、それぞれのエラーが発生した場合にエラーが発生するまでの成功数/失敗数を取得するようにする。

    - 処理

        - 基本的な流れはelasticsearch.helpersのbulk()と同様。<br>
        　streaming_bulk()を実行し、1件ずつsuccessとfailedをカウントしていく。
        - 独自例外クラスをConnectionError、ConnectionTimeout、Exception分作成し、reidex_bulk()で例外が発生した場合にそれぞれキャッチする。
            - そこまでのsuccess、failed、errors、エラー内容を取得し、raiseで呼び出し元へ返す。

    - 戻り値

        - successとfail、もしくはsuccessとerrors


    ### 3. BulkBaseException(Exception)

    - 引数

        - Exception：例外エラー

    - 処理

        - 成功、失敗、エラー、例外オブジェクトを取得する。

    ### 4. BulkConnectionError(BulkBaseException, ConnectionError)/BulkConnectionTimeout(BulkBaseException, ConnectionTimeout)/BulkException(BulkBaseException)

    - 引数

        - BulkBaseException
        - ConnectionError、ConnectionTimeout

    - 処理

        - 3. BulkBaseExceptionと同様の処理を行う。

## 更新履歴

| 日付       | GitHubコミットID                           | 更新内容                                                 |
| ---------- | ------------------------------------------ | -------------------------------------------------------- |
| 2025/09/ |         | 初版作成      