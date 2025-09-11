# Reindex 不具合修正

## 目次
- [count誤差修正](#count誤差修正)
    - [bulk()対象件数の取得](#bulk対象件数の取得)
    - [bulk()処理成功(エラー発生なし)](#bulk処理成功エラー発生なし)
    - [例外エラー発生(BulkIndexError)](#例外エラー発生bulkindexerror)
    - [例外エラー発生(BulkIndexError以外)](#例外エラー発生bulkindexerror以外)
    - [countの出力](#countの出力)


## count誤差修正

- 概要

    例外エラー時、出力されるcountの誤差をなくすための修正

- 修正方針

    - process_bulk_queue()をリファクタリングした新たな関数を作成することにより、count誤差が起きないようにする。<br>
    そのため、新たな関数の実装はprocess_bulk_queue()を元にして進める。
    - 例外エラーが発生した場合、リトライせずにエラーとする。
    - エラー発生時にbulkを最後まで実行する設定の場合でも、ログとしてアイテムごとの結果やエラー内容を出力する

- 修正内容

    ### bulk()対象件数の取得

    未実施件数を正確に把握するため、関数の頭でmessages_countを宣言し、messagesを取得したタイミングでlenを取得する。

        （省略）
        self.count = 0
        messages_count = 0 # 追加
        （中略）
        messages = list(consumer.iterqueue())
        messages_count = len(messages) # 追加

    ### bulk()処理成功(エラー発生なし)

    successおよびfailを加算代入していたが、そのまま代入する。

        # 修正前
        success = success + _success
        fail = fail + _fail

        # 修正後
        success = _success
        fail = _fail

    ### 例外エラー発生(BulkIndexError)
    
    1. 成功、失敗、未実施数を計算する。<br>BulkIndexErrorはerrorsにエラー情報が集約されるため、errorsからfailを取得する。
    2. エラーが起きたID,およびエラー原因を出力する。
    3. 再bulk()しないため、error_ids = [] 以降の処理を削除する。

            # 修正後
            except BulkIndexError as be:
                with conn.channel() as chan:
                    name, af_queues_cnt, consumers = chan.queue_declare(queue=current_app.config['INDEXER_MQ_ROUTING_KEY'], passive=True)
                    current_app.logger.debug("name:{}, queues:{}, consumers:{}".format(name, af_queues_cnt, consumers))
                    fail = len(be.errors)
                    success = self.count - fail
                    unprocessed = messages_count - self.count
                    for error in be.errors:
                        click.secho("{}, {}".format(error['index']['_id'],error['index']['error']['type']),fg='red')

    ### 例外エラー発生(BulkIndexError以外)

    - bulk()中に例外エラーが発生した場合、エラーが発生する前までに成功したデータの登録は完了している。<br>
    しかしbulk()を完走しなければ成功数/失敗数の取得は出来ない。<br>
    bulk()の代わりに以下で作成するstreaming_bulk()のラップ関数を呼び出すようにする。<br>
    例外エラークラスを作成し、それぞれのエラーが発生した場合にエラーが発生するまでの成功数/失敗数を取得するようにする。

            # 新規作成(関数名は仮称)
            def reindex_bulk(self, client, actions, stats_only=False, *args, **kwargs):
                success, failed = 0, 0
                errors = []
                ignore_status = kwargs.pop('ignore_status', None)
                span_name = kwargs.pop('span_name', None)
                kwargs.pop('yield_ok', None)
                if 'span_name' in kwargs:
                    del kwargs['span_name']
                if 'ignore_status' in kwargs:
                    del kwargs['ignore_status']

                try:
                    streaming_bulk_args = [client, actions]
                    streaming_bulk_kwargs = {"yield_ok": True}
                    if span_name is not None:
                        streaming_bulk_kwargs["span_name"] = span_name
                    if ignore_status is not None:
                        streaming_bulk_kwargs["ignore_status"] = ignore_status

                    for ok, item in streaming_bulk(
                        *streaming_bulk_args,
                        *args,
                        **streaming_bulk_kwargs,
                        **kwargs
                    ):
                        if not ok:
                            if not stats_only:
                                errors.append(item)
                            failed += 1
                        else:
                            success += 1
                    return (success, failed) if stats_only else (success, errors)
                except BulkIndexError as e:
                    raise
                except (ConnectionError, ConnectionTimeout, Exception) as e:
                    if isinstance(e, ConnectionError):
                        raise BulkConnectionError(success, failed, errors, e)
                    elif isinstance(e, ConnectionTimeout):
                        raise BulkConnectionTimeout(success, failed, errors, e)
                    else:
                        raise BulkException(success, failed, errors, e)

    ① ConnectionError

    1. 成功、失敗、未実施数を計算する。

    2. エラーが起きたID、およびエラー原因を出力する。

    3. 再bulk()しないため、error_ids = [] 以降の処理は不要。

            # 修正後
            except (BulkConnectionError, ConnectionError) as ce:
                with conn.channel() as chan:
                    name, af_queues_cnt, consumers = chan.queue_declare(queue=current_app.config['INDEXER_MQ_ROUTING_KEY'], passive=True)
                    current_app.logger.debug("name:{}, queues:{}, consumers:{}".format(name, af_queues_cnt, consumers))
                    if '_success' in locals() or '_fail' in locals():
                        success = _success
                        fail = _fail
                        errors = []
                        if isinstance(fail, list):
                            errors = fail
                    else:
                        success = ce.success if hasattr(ce, 'success') else 0
                        fail = ce.failed if hasattr(ce, 'failed') else 0
                        errors = ce.errors if hasattr(ce, 'errors') else []
                    if len(errors) > 0:
                        for error in errors:
                            click.secho("{}, {}".format(error['index']['_id'],error['index']['error']['type']),fg='red')
                        fail = len(errors)
                    unprocessed = messages_count - (success + fail) if messages_count > (success + fail) else 0

    ② ConnectionTimeout

    1. 成功、失敗、未実施数の計算を追加する。

    2. エラーが起きたID,およびエラー原因を出力する。

            # 修正後
            except (BulkConnectionTimeout, ConnectionTimeout) as ce:
                click.secho("Error: {}".format(ce),fg='red')
                click.secho("INDEXER_BULK_REQUEST_TIMEOUT: {} sec".format(req_timeout),fg='red')
                click.secho("Please change value of INDEXER_BULK_REQUEST_TIMEOUT and retry it.",fg='red')
                click.secho("processing: {}".format(self.count),fg='red')
                click.secho("latest processing id: {}".format(self.latest_item_id),fg='red')
                if '_success' in locals() or '_fail' in locals():
                    success = _success
                    fail = _fail
                    errors = []
                    if isinstance(fail, list):
                        errors = fail
                else:
                    success = ce.success if hasattr(ce, 'success') else 0
                    fail = ce.failed if hasattr(ce, 'failed') else 0
                    errors = ce.errors if hasattr(ce, 'errors') else []
                if len(errors) > 0:
                    for error in errors:
                        click.secho("{}, {}".format(error['index']['_id'],error['index']['error']['type']),fg='red')
                    fail = len(errors)
                unprocessed = messages_count - (success + fail) if messages_count > (success + fail) else 0

    ③ Exception

    1. 成功、失敗、未実施数の計算を追加する。

    2. エラーが起きたID,およびエラー原因を出力する。

            # 修正後
            except (BulkException, Exception) as e:
                current_app.logger.error(e)
                current_app.logger.error(traceback.format_exc())
                if '_success'  in locals() or '_fail' in locals():
                    success = _success
                    fail = _fail
                    errors = []
                    if isinstance(fail, list):
                        errors = fail
                else:
                    success = e.success if hasattr(e, 'success') else 0
                    fail = e.failed if hasattr(e, 'failed') else 0
                    errors = e.errors if hasattr(e, 'errors') else []
                if len(errors) > 0:
                    for error in errors:
                        click.secho("{}, {}".format(error['index']['_id'],error['index']['error']['type']),fg='red')
                    fail = len(errors)
                unprocessed = messages_count - (success + fail) if messages_count > (success + fail) else 0

    ### countの出力

    - 未実施項目がある場合はcount出力にunprocessedを追加する。

            # 修正後
            if unprocessed == 0:
                count = (success,fail)
                click.secho("count(success, error): {}".format(count),fg='green')
            else:
                count = (success,fail,unprocessed)
                click.secho("count(success, error, unprocessed): {}".format(count),fg='green')
            return count


## 更新履歴

| 日付       | GitHubコミットID                           | 更新内容                                                 |
| ---------- | ------------------------------------------ | -------------------------------------------------------- |
| 2025/09/ |         | 初版作成      