# ジョブキュー実装概要

このドキュメントでは Dify プラットフォームにおけるジョブキューの仕組みと実装詳細をまとめます。

## Celery による非同期処理

バックグラウンドタスクは Celery を用いて実装されています。`api/extensions/ext_celery.py` では Flask アプリケーションと連携した Celery インスタンスを生成し、Redis ブローカーや結果バックエンド、Beat スケジュールが設定されています。

```python
# api/extensions/ext_celery.py 抜粋
```
以下に初期化処理の一部を示します。

```python
def init_app(app: DifyApp) -> Celery:
    class FlaskTask(Task):
        def __call__(self, *args: object, **kwargs: object) -> object:
            with app.app_context():
                return self.run(*args, **kwargs)

    broker_transport_options = {}

    if dify_config.CELERY_USE_SENTINEL:
        broker_transport_options = {
            "master_name": dify_config.CELERY_SENTINEL_MASTER_NAME,
            "sentinel_kwargs": {
                "socket_timeout": dify_config.CELERY_SENTINEL_SOCKET_TIMEOUT,
            },
        }

    celery_app = Celery(
        app.name,
        task_cls=FlaskTask,
        broker=dify_config.CELERY_BROKER_URL,
        backend=dify_config.CELERY_BACKEND,
        task_ignore_result=True,
    )
```

Beat スケジュールも同ファイルで設定されており、定期実行タスクが登録されています。

```python
    celery_app.set_default()
    app.extensions["celery"] = celery_app

    imports = [
        "schedule.clean_embedding_cache_task",
        "schedule.clean_unused_datasets_task",
        "schedule.create_tidb_serverless_task",
        "schedule.update_tidb_serverless_status_task",
        "schedule.clean_messages",
        "schedule.mail_clean_document_notify_task",
    ]
    day = dify_config.CELERY_BEAT_SCHEDULER_TIME
    beat_schedule = {
        "clean_embedding_cache_task": {
            "task": "schedule.clean_embedding_cache_task.clean_embedding_cache_task",
            "schedule": timedelta(days=day),
        },
        "clean_unused_datasets_task": {
            "task": "schedule.clean_unused_datasets_task.clean_unused_datasets_task",
            "schedule": timedelta(days=day),
        },
        "create_tidb_serverless_task": {
            "task": "schedule.create_tidb_serverless_task.create_tidb_serverless_task",
            "schedule": crontab(minute="0", hour="*"),
        },
        "update_tidb_serverless_status_task": {
            "task": "schedule.update_tidb_serverless_status_task.update_tidb_serverless_status_task",
            "schedule": timedelta(minutes=10),
        },
        "clean_messages": {
            "task": "schedule.clean_messages.clean_messages",
            "schedule": timedelta(days=day),
        },
        # every Monday
        "mail_clean_document_notify_task": {
            "task": "schedule.mail_clean_document_notify_task.mail_clean_document_notify_task",
            "schedule": crontab(minute="0", hour="10", day_of_week="1"),
        },
    }
    celery_app.conf.update(beat_schedule=beat_schedule, imports=imports)

```

ワーカーは Docker イメージ内の `entrypoint.sh` で起動します。

```bash
if [[ "${MODE}" == "worker" ]]; then

  # Get the number of available CPU cores
  if [ "${CELERY_AUTO_SCALE,,}" = "true" ]; then
    # Set MAX_WORKERS to the number of available cores if not specified
    AVAILABLE_CORES=$(nproc)
    MAX_WORKERS=${CELERY_MAX_WORKERS:-$AVAILABLE_CORES}
    MIN_WORKERS=${CELERY_MIN_WORKERS:-1}
    CONCURRENCY_OPTION="--autoscale=${MAX_WORKERS},${MIN_WORKERS}"
  else
    CONCURRENCY_OPTION="-c ${CELERY_WORKER_AMOUNT:-1}"
  fi

  exec celery -A app.celery worker -P ${CELERY_WORKER_CLASS:-gevent} $CONCURRENCY_OPTION \
    --max-tasks-per-child ${MAX_TASK_PRE_CHILD:-50} --loglevel ${LOG_LEVEL:-INFO} \
    -Q ${CELERY_QUEUES:-dataset,mail,ops_trace,app_deletion}
```

### 主なキュー

- **dataset**: データセットのインデックス処理や削除処理など、データ管理系のタスク
- **mail**: 各種メール送信
- **ops_trace**: アプリの操作ログ（トレース）送信
- **app_deletion**: アプリ削除に伴うデータのクリーンアップ

タスク定義例を以下に示します。

```python
@shared_task(queue="dataset")
def delete_account_task(account_id):
    account = db.session.query(Account).filter(Account.id == account_id).first()
    try:
        BillingService.delete_account(account_id)
```

```python
@shared_task(queue="mail")
def send_invite_member_mail_task(language: str, to: str, token: str, inviter_name: str, workspace_name: str):
    """
    Async Send invite member mail
    :param language
    :param to
```

```python

@shared_task(queue="ops_trace")
def process_trace_tasks(file_info):
    """
    Async process trace tasks
    Usage: process_trace_tasks.delay(tasks_data)
    """
    from core.ops.ops_trace_manager import OpsTraceManager
```

```python
@shared_task(queue="app_deletion", bind=True, max_retries=3)
def remove_app_and_related_data_task(self, tenant_id: str, app_id: str):
    logging.info(click.style(f"Start deleting app and related data: {tenant_id}:{app_id}", fg="green"))
    start_at = time.perf_counter()
    try:
        # Delete related data
```

## チャットメッセージ生成のキュー管理

Celery とは別に、チャットやワークフローの実行では `AppQueueManager` を基底とした独自キューを利用します。イベントは `queue.Queue` に保持され、クライアントは `listen` メソッドでリアルタイムに受信します。

```python
    TASK_PIPELINE = 2


class AppQueueManager:
    def __init__(self, task_id: str, user_id: str, invoke_from: InvokeFrom) -> None:
        if not user_id:
            raise ValueError("user is required")

        self._task_id = task_id
        self._user_id = user_id
        self._invoke_from = invoke_from

        user_prefix = "account" if self._invoke_from in {InvokeFrom.EXPLORE, InvokeFrom.DEBUGGER} else "end-user"
        redis_client.setex(
            AppQueueManager._generate_task_belong_cache_key(self._task_id), 1800, f"{user_prefix}-{self._user_id}"
        )

        q: queue.Queue[WorkflowQueueMessage | MessageQueueMessage | None] = queue.Queue()

        self._q = q

    def listen(self):
        """
        Listen to queue
        :return:
        """
        # wait for APP_MAX_EXECUTION_TIME seconds to stop listen
        listen_timeout = dify_config.APP_MAX_EXECUTION_TIME
        start_time = time.time()
        last_ping_time: int | float = 0
        while True:
            try:
                message = self._q.get(timeout=1)
                if message is None:
                    break

                yield message
            except queue.Empty:
                continue
            finally:
                elapsed_time = time.time() - start_time
                if elapsed_time >= listen_timeout or self._is_stopped():
                    # publish two messages to make sure the client can receive the stop signal
                    # and stop listening after the stop signal processed
                    self.publish(
                        QueueStopEvent(stopped_by=QueueStopEvent.StopBy.USER_MANUAL), PublishFrom.TASK_PIPELINE
                    )

                if elapsed_time // 10 > last_ping_time:
                    self.publish(QueuePingEvent(), PublishFrom.TASK_PIPELINE)
                    last_ping_time = elapsed_time // 10
```

チャットアプリでは `MessageBasedAppQueueManager` と専用スレッドによりメッセージ生成を行います。

```python
class MessageBasedAppQueueManager(AppQueueManager):
    def __init__(
        self, task_id: str, user_id: str, invoke_from: InvokeFrom, conversation_id: str, app_mode: str, message_id: str
    ) -> None:
        super().__init__(task_id, user_id, invoke_from)

        self._conversation_id = str(conversation_id)
        self._app_mode = app_mode
        self._message_id = str(message_id)

    def construct_queue_message(self, event: AppQueueEvent) -> QueueMessage:
        return MessageQueueMessage(
            task_id=self._task_id,
            message_id=self._message_id,
            conversation_id=self._conversation_id,
            app_mode=self._app_mode,
            event=event,
        )

```

```python
            conversation_id=conversation.id,
            app_mode=conversation.mode,
            message_id=message.id,
        )

        # new thread
        worker_thread = threading.Thread(
            target=self._generate_worker,
            kwargs={
                "flask_app": current_app._get_current_object(),  # type: ignore
                "application_generate_entity": application_generate_entity,
                "queue_manager": queue_manager,
                "conversation_id": conversation.id,
                "message_id": message.id,
            },
        )

```

## Ops Trace キュー

操作ログは `TraceQueueManager` が一旦キューに溜め、一定間隔で Celery タスク `process_trace_tasks` へ送信します。

```python

class TraceQueueManager:
    def __init__(self, app_id=None, user_id=None):
        global trace_manager_timer

        self.app_id = app_id
        self.user_id = user_id
        self.trace_instance = OpsTraceManager.get_ops_trace_instance(app_id)
        self.flask_app = current_app._get_current_object()  # type: ignore
        if trace_manager_timer is None:
            self.start_timer()

    def add_trace_task(self, trace_task: TraceTask):
        global trace_manager_timer, trace_manager_queue
        try:
            if self.trace_instance:
                trace_task.app_id = self.app_id
                trace_manager_queue.put(trace_task)
        except Exception as e:
            logging.exception(f"Error adding trace task, trace_type {trace_task.trace_type}")
        finally:
            self.start_timer()

    def collect_tasks(self):
        global trace_manager_queue
        tasks: list[TraceTask] = []
        while len(tasks) < trace_manager_batch_size and not trace_manager_queue.empty():
            task = trace_manager_queue.get_nowait()
            tasks.append(task)
            trace_manager_queue.task_done()
        return tasks

    def run(self):
        try:
            tasks = self.collect_tasks()
            if tasks:
                self.send_to_celery(tasks)
        except Exception as e:
            logging.exception("Error processing trace tasks")

    def start_timer(self):
        global trace_manager_timer
        if trace_manager_timer is None or not trace_manager_timer.is_alive():
            trace_manager_timer = threading.Timer(trace_manager_interval, self.run)
            trace_manager_timer.name = f"trace_manager_timer_{time.strftime('%Y-%m-%d %H:%M:%S', time.localtime())}"
            trace_manager_timer.daemon = False
            trace_manager_timer.start()

    def send_to_celery(self, tasks: list[TraceTask]):
        with self.flask_app.app_context():
            for task in tasks:
```

```python
            for task in tasks:
                if task.app_id is None:
                    continue
                file_id = uuid4().hex
                trace_info = task.execute()
                task_data = TaskData(
                    app_id=task.app_id,
                    trace_info_type=type(trace_info).__name__,
                    trace_info=trace_info.model_dump() if trace_info else None,
                )
                file_path = f"{OPS_FILE_PATH}{task.app_id}/{file_id}.json"
                storage.save(file_path, task_data.model_dump_json().encode("utf-8"))
                file_info = {
                    "file_id": file_id,
                    "app_id": task.app_id,
                }
                process_trace_tasks.delay(file_info)
```

以上のように、Celery を利用したバックグラウンド処理と、アプリ専用のキュー管理が組み合わさることで、Dify のジョブ処理は効率的に行われています。
