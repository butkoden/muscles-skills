# Muscles Data: политики, SQL и жизненный цикл

Этот справочник применяй, когда данные нужно ограничивать контекстом запроса, связывать между независимыми пакетами или сопровождать транзакционными действиями. Перед использованием сверяй API с установленными версиями `muscles-data` и `muscles-data-sql`: доступность контракта зависит от версии.

## Выбор механизма

1. Приложение аутентифицирует субъекта, проверяет его доступ к области и создаёт новый `DataOperationContext` с доверенными атрибутами на запрос. ID из URL или тела запроса сам по себе не подтверждает доступ.
2. Пакет объявляет тип через `runtime.register_object_type(target, object_type, owner_field=...)`. `owner_field` — метаданные; автоматического фильтра по владельцу нет.
3. Независимый модуль регистрирует политику в `runtime.policies` и обработчик в `runtime.object_lifecycle`. Пакет объектов не импортирует модуль, управляющий областями. Несколько политик объединяются через AND; явный отказ вызывает `DataPolicyDeniedError`. Пустой pipeline не ограничивает доступ.
4. В SQL применяй `native_repository_transaction` или `native_repository_from_runtime`; они используют политики и обработчики runtime. При отключении модуля политики предусмотри отдельную проверку владельца или иной обязательный контроль.

`DataRelatedExists` ограничивает видимость через связанную таблицу. Для `CREATE` нельзя требовать заранее существующую ссылку: её добавляет обработчик после создания объекта в той же транзакции. `UPSERT` проверяет условия и на ветке создания; если политика не может безопасно проверить эту ветку, явно запрети операцию. `replaces_owner_scope` имеет смысл только для адаптера, который отдельно реализует ограничение по владельцу; сам флаг не фильтрует данные и требует предикат. `NativePolicyRepository` не добавляет фильтр владельца автоматически.

## Рабочий пример: доступ и события

Ниже автономный SQLite-пример. В реальном приложении используй существующий `app.resource`, а не создавай второй runtime. Пример предполагает, что приложение заранее подтвердило право пользователя работать в `scope_id`; это не реализация аутентификации или проверки членства.

```python
"""Runnable example of scoped objects and transactional lifecycle events."""
from muscles.data.catalog import DataAdapterCatalog
from muscles.data.config import DataConfig
from muscles.data.policies import (
    DataOperationContext,
    DataOperationKind,
    DataPolicyDecision,
    DataPredicate,
    DataPredicateOperator,
    DataRelatedExists,
)
from muscles.data.ports import SqlTransactionPort
from muscles.data.runtime import DataRuntime
from muscles.data.sql import native_repository_transaction
from muscles.data.sql.data_adapter import NativeSqlFactory


class ScopePolicy:
    def evaluate(self, operation):
        if operation.object_type != "demo:document":
            return DataPolicyDecision()
        scope = operation.context.attributes.get("scope_id")
        if not operation.context.subject_uid or not scope:
            return DataPolicyDecision(allowed=False, reason="Authenticated scope required")
        if operation.kind is DataOperationKind.UPSERT:
            return DataPolicyDecision(allowed=False, reason="Use create or update")
        if operation.kind is DataOperationKind.CREATE:
            return DataPolicyDecision()
        return DataPolicyDecision(predicates=(DataPredicate(
            "uid", DataPredicateOperator.RELATED_EXISTS,
            DataRelatedExists("scope_links", "object_uid", {
                "scope_id": scope, "object_type": "demo:document",
            }),
        ),))


def maintain_links(transaction, event):
    if event.object_type != "demo:document":
        return
    if event.action == "created":
        transaction.execute(
            "INSERT INTO scope_links (scope_id, object_type, object_uid) VALUES (?, ?, ?)",
            (event.context.attributes["scope_id"], event.object_type, event.object_uid),
        )
    elif event.action == "deleted":
        transaction.execute(
            "DELETE FROM scope_links WHERE object_type = ? AND object_uid = ?",
            (event.object_type, event.object_uid),
        )


def reject_example(transaction, event):
    if event.action == "created" and event.object_uid == "broken":
        raise RuntimeError("Example handler failed")


def main():
    catalog = DataAdapterCatalog.with_defaults()
    catalog.register(NativeSqlFactory())
    config = DataConfig.from_raw({"resources": {
        "database": {"type": "native_sql", "driver": "sqlite", "path": ":memory:"},
    }})
    runtime = DataRuntime(config=config, catalog=catalog)
    try:
        sql = runtime.connect("database", SqlTransactionPort)
        with sql.transaction() as tx:
            tx.execute("CREATE TABLE documents (uid TEXT PRIMARY KEY, title TEXT NOT NULL)")
            tx.execute("""CREATE TABLE scope_links (
                scope_id TEXT NOT NULL, object_type TEXT NOT NULL, object_uid TEXT NOT NULL,
                PRIMARY KEY (scope_id, object_type, object_uid))""")

        runtime.register_object_type("documents", "demo:document")
        runtime.policies.register(ScopePolicy())
        runtime.object_lifecycle.register(maintain_links)
        runtime.object_lifecycle.register(reject_example)

        first = DataOperationContext(subject_type="user", subject_uid="user-1", attributes={"scope_id": "one"})
        second = DataOperationContext(subject_type="user", subject_uid="user-1", attributes={"scope_id": "two"})
        with native_repository_transaction(runtime, "database", "documents", context=first, placeholder="?") as repo:
            repo.create({"uid": "doc-1", "title": "Example"})
            assert repo.get("doc-1")["title"] == "Example"
        with native_repository_transaction(runtime, "database", "documents", context=second, placeholder="?") as repo:
            assert repo.get("doc-1") is None
            assert repo.update("doc-1", {"title": "Not allowed"}) == 0

        try:
            with native_repository_transaction(runtime, "database", "documents", context=first, placeholder="?") as repo:
                repo.create({"uid": "broken", "title": "Rolled back"})
        except RuntimeError as exc:
            assert str(exc) == "Example handler failed"
        else:
            raise AssertionError("The failing handler should abort the transaction")
        with sql.transaction() as tx:
            assert tx.fetch_one("SELECT uid FROM documents WHERE uid = ?", ("broken",)) is None
            assert tx.fetch_one("SELECT object_uid FROM scope_links WHERE object_uid = ?", ("broken",)) is None

        with native_repository_transaction(runtime, "database", "documents", context=first, placeholder="?") as repo:
            assert repo.delete("doc-1") == 1
        with sql.transaction() as tx:
            assert tx.fetch_one("SELECT object_uid FROM scope_links WHERE object_uid = ?", ("doc-1",)) is None
        print("Scoped access and transactional lifecycle: OK")
    finally:
        runtime.close()


if __name__ == "__main__":
    main()
```

В установленном окружении этот пример можно запустить как самостоятельный Python-файл. Он проверяет доступ из другой области, откат объекта и связи при ошибке обработчика и очистку связей при удалении. В PostgreSQL вместо `?` используй `%s` и соответствующую конфигурацию драйвера. Не подставляй в SQL идентификаторы или фрагменты из недоверенного ввода.

## Границы SQL-адаптера

- `native_repository_transaction(runtime, resource, table, context=...)` открывает транзакцию; `native_repository_from_runtime(runtime, transaction, table, context=...)` использует уже открытую. Зарегистрированный тип определяется автоматически; несовпадающий явно заданный тип отклоняется.
- `get` возвращает `None` для отсутствующей или скрытой строки; `update` и `delete` возвращают число затронутых строк. Списки, `count`, агрегаты и пагинация применяют политики до выдачи результатов. При `list_joined` проверяй политики обеих таблиц и передавай `related_object_type` для связанного типа.
- Проверяй доступ к обоим концам связи. `bulk_create`, `bulk_update` и `upsert` используют savepoint и проверки записей, но общий коммит принадлежит внешней транзакции. Не считай savepoint отдельным коммитом. Ошибка обработчика транзакционного события отменяет запись; не подавляй её, пытаясь продолжить фиксацию.
- `tx.execute`, `fetch_one` и `fetch_all` не применяют политики и не отправляют события автоматически. Если без специального SQL не обойтись, вычисли ограничение и включи его в реальный запрос вместе с параметрами. При создании отдельно проверь предикаты через `validate_create_predicates`; при ручной отправке используй `tx.emit_object_event` в той же транзакции. Каждый JOIN и изменение связанных данных также должны быть защищены.

```python
from muscles.data.policies import DataOperationKind, DataOperationRequest
from muscles.data.sql.native_policy import compile_operation_scope

# runtime, context и tx существуют; запрос здесь использует PostgreSQL.
operation = DataOperationRequest(
    DataOperationKind.READ, "documents", context, object_type="demo:document",
)
clause, parameters = compile_operation_scope(
    runtime.policies, operation, target_alias="d", placeholder="%s",
)
query = 'SELECT "d".* FROM "documents" AS "d"'
if clause:
    query += f" WHERE {clause}"
rows = tx.fetch_all(query, parameters)
```

`ObjectLifecycleEvent` — синхронный вызов в транзакции, а не `EventPublisherPort`, очередь или журнал. Обновление и чтение сами по себе не создают таких событий; `upsert` отправляет `created` только при вставке. Для внешней доставки проектируй отдельный надёжный механизм, например outbox. `DataPolicyPipeline.explain(operation)` показывает цель и операторы без значений, но ничего не защищает. `DataNonAtomicPolicyMode.ALLOW_PRECHECK` ослабляет защиту: между проверкой и записью возможна гонка; применяй его только с осознанным решением приложения.

## Миграция и проверка

Старые данные автоматически не получают новые связи или владельцев. Переноси привязки повторяемо и транзакционно, с проверкой отсутствующих объектов и сохранением исходной области; проверяй данные до включения ограничения. Удаление объекта должно очищать связанные записи в той же транзакции, а не через прямую зависимость пакета объектов от пакета областей.

Для SQL-миграций по умолчанию добавляй новую ревизию. Если конкретная согласованная совместимая правка уже применённой миграции действительно необходима, текущий backend поддерживает `compatible_checksums` с точными прежними суммами. Это не разрешает игнорировать остальные изменения истории. Проверь установку с нуля, существующую историю, оба поддерживаемых диалекта и сохранность данных.

Минимальная проверка поведения: разные пользователи и области; отсутствие необходимого контекста; общая карточка в нескольких областях; чужие связанные объекты; создание, чтение, изменение и удаление; пагинация и JOIN; rollback объекта и ссылки при ошибке; повторный перенос старых данных; работа без необязательного модуля политики с явной защитой владельца. Для адаптеров с неатомарной записью проверяй режим отказа и отдельно обосновывай любые предварительные проверки.

При наличии исходников библиотек см. их актуальные руководства `muscles-data/docs/data-policies.ru.md` и `muscles-data-sql/docs/data-policies.ru.md` и тесты соответствующей версии. SQLAlchemy не является частью `muscles-data-sql`.
