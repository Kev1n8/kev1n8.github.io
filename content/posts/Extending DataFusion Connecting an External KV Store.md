+++
title = "Extending DataFusion: Connecting an External KV Store"
date = 2024-09-13

[taxonomies]
tags = ["rust", "datafusion"]
+++



## 扩展Datafusion之从外部KV存储

作为library，Datafusion在各个层次都提供了Extension trait，实现这些trait可以扩展datafusion的功能，非常方便。本文主要介绍Datafusion提供的`TableProvider`, `ExecutionPlan` 和 `DataSink`三个`trait`。通过这几个`trait`我们可以实现让只能在内存(`datafusion::datasource::MemTable`)中的数据存储到任何我们想要的位置。

## TableProvider

`TableProvider`顾名思义，让一个结构具有提供表相关功能的能力，一系列方法中最核心的就是要实现`scan`方法（当然，还有`insert_into`方法）。

一般来说，要让自定义的数据源来`TableProvider`。在我的实现中，我把KV存储作为`TableProvider`，使用的是RocksDB，那么相关的结构体就是这样：

```rust
#[derive(Debug, Clone)]
pub struct KVTable {
    pub db: Arc<DB>,
    pub table_id: u64,
    pub meta: KVTableMetaRef,
}
```

我的实现比较简陋，只有一些必要的成员。接下来为了实现扫描功能，这个`TableProvider`需要有一个`scan`函数（和`insert_into`函数，如果想要支持写入到KV的话），返回`SendableStream`供后续异步回调数据。

但是实际在做的时候呢，并不会直接把执行细节放在数据源这里，而是通过构造一个`ExecutionPlan`来执行。

```rust
#[async_trait]
impl TableProvider for KVSource {
...
    async fn scan(
        &self,
        _state: &dyn Session,
        projection: Option<&Vec<usize>>,
        _filters: &[Expr],
        _limit: Option<usize>
    ) -> Result<Arc<dyn ExecutionPlan>> {
        self.create_scan_physical_plan(
            self.table_id,
            projection,
            self.schema()
        ).await
    }
...
}
```

## ExecutionPlan

一个`ExecutionPlan`的职责，就是提供一个`execute`方法，能够返回`SendableStream`。而这个`Stream`会回调`ExecutionPlan`内部的一个方法，这个被回调的方法就是真正的功能实现的地方。以`DBTableScanExec`为例：

```rust
#[derive(Debug)]
pub struct DBTableScanExec {
    id: u64,
    db: Arc<DB>,
    schema: SchemaRef,
    cache: PlanProperties,
}
```

我的初步实现按照以下的想法和限制：

1. 数据类型只支持String，便于统一`put`和`get`
2. key的编码方式是 `t[table_id]_c[col_name]_r[row_id]`
3. 暂不支持`null`

那么就可以有下面的`read_columns`方法：

```rust
    fn read_columns(&self) -> Vec<ArrayRef> {
        let mut result: Vec<ArrayRef> = Vec::new();

        for col in &self.schema.fields {
            // Temporary ugly code
            let start_key = format!("t{}_c{}_r000001", self.id, col.name());
            let prefix = format!("t{}_c{}", self.id, col.name());

            let mut iter = self.db.iterator(IteratorMode::From(start_key.as_bytes(), rocksdb::Direction::Forward));
            let mut values = Vec::new();

            while let Some(Ok((key, value))) = iter.next() {
                // Safety:
                // All keys and values are supposed to be encoded from utf8
                unsafe {
                    let key_str = String::from_utf8_unchecked(key.to_vec());
                    if !key_str.starts_with(prefix.as_str()) {
                        break;
                    }
                    let value_str = String::from_utf8_unchecked(value.to_vec());
                    values.push(value_str);
                }
            }

            let array: ArrayRef = Arc::new(StringArray::from(values));
            result.push(array);
        }
        result
    }
```

这样一来，就可以查本`DBTableScanExec`实例对应`table_id`和`schema`的所有数据了。

## DataSink

`DataSink`的特点是有一个`write_all`方法，它的参数提供了一个`Stream`表示数据源，我们要实现的则是将`Stream`带来的数据写入"Sink"中，可以是各种目标，我这里的实现是将数据放进KV存储引擎中。

首先说明，实现`CustomDataSink`的目的是让Datafusion的`DataSinkExec`能够执行自定义的`write_all`方法。这个流程，举例来说，先在`ctx`中注册了已经实现了`KVTable`，然后使用`LogicPlanBuilder`构建一个简单的`insert_into` 逻辑计划。这时候`KVTable`中实现的`insert_into`就派上用场了，它会返回一个`DataSinkExec`附带了目标Sink的信息（也就是我们实现的`KVSink`）。再用`ctx`生成执行计划，最后调用`collect`回调计算函数，此时`write_all`就会在数据准备好之后被调用。

下面来看关键代码：

```rust
#[async_trait]
impl TableProvider for KVSource {
...
    async fn insert_into(
        &self,
        _state: &dyn Session,
        input: Arc<dyn ExecutionPlan>,
        _overwrite: bool
    ) -> Result<Arc<dyn ExecutionPlan>> {
        let sink = Arc::new(KVTableSink::new(self.table_id, &self.db));
        Ok(Arc::new(DataSinkExec::new(
            input,  // input source informed by the caller of insert_into
            sink,   // target sink
            self.meta.schema.clone(),
            None,
        )))
    }
...
}
```

这个函数帮助我们构造执行计划，`DataSinkExec`会在执行的时候调用`write_all`，参考下面的`DataSinkExec::execute`：

```rust
impl ExecutionPlan for DataSinkExec {
...
	fn execute(
        &self,
        partition: usize,
        context: Arc<TaskContext>,
    ) -> Result<SendableRecordBatchStream> {
        if partition != 0 {
            return internal_err!("DataSinkExec can only be called on partition 0!");
        }
        let data = execute_input_stream(  // Fetch input stream here
            Arc::clone(&self.input),
            Arc::clone(&self.sink_schema),
            0,
            Arc::clone(&context),
        )?;

        let count_schema = Arc::clone(&self.count_schema);
        let sink = Arc::clone(&self.sink);

        let stream = futures::stream::once(async move {
            sink.write_all(data, &context).await.map(make_count_batch)  // write_all here
        })
        .boxed();

        Ok(Box::pin(RecordBatchStreamAdapter::new(
            count_schema,
            stream,
        )))
    }
...
}
```

我的实现没有考虑context要求和多个partition的情况，只实现了基本的写入。其中`put_batch_into_db`的实现细节就不展开了，大致就是将一个`cell`（不是很高效的value单位）对应一个`key`（编码方式大概是：`t[table_id]_c[col_id]_r[row_id]`）存入rocksdb。下面是我的`write_all`实现：

```rust
	async fn write_all(
        &self,
        data: SendableRecordBatchStream,
        _context: &Arc<TaskContext>,
    ) -> Result<u64> {
        let mut data = data;
        let mut cnt = 0;
        if let Some(batch) = data.next().await.transpose()? {
            cnt = self.put_batch_into_db(&batch).await?;
        }
        Ok(cnt)
    }
```

## BONUS：测试

另外补充一下写的几个基本测试，主要是记录一下测试的方法。

首先，是元数据的编解码测试：

```rust
    #[tokio::test]
    async fn test_meta_encode_decode() {
        let meta = KVTableMeta {
            id: 1002,
            name: "TableTest".to_string(),
            schema: Arc::new(Schema::new(Fields::from(vec![
                Field::new("column1", DataType::Utf8, false),
                Field::new("column2", DataType::Utf8, false),
            ]))),
            highest: 0,
        };
        let key = meta.make_key();
        let val = meta.make_value();

        assert_eq!(key, "mt1002".to_string().into_bytes());
        assert_eq!(val, "t1002_TableTest_0_c2_column1_column2".to_string().into_bytes());

        let decode = KVTableMeta::from(val);
        assert_eq!(meta.id, decode.id);
        assert_eq!(meta.highest, decode.highest);
        assert_eq!(meta.name, decode.name);
    }
```

对于`DBTableScanExec`结构，定义表元数据、直接通过KV的API添加100个元素，然后使用`DBTableScanExec`执行返回的`Stream`获取`Batch`后逐行比对：

```rust
    #[tokio::test]
    async fn test_read_from_db() {
        let meta = Arc::new(KVTableMeta {
            id: 101,
            name: "a".to_string(),
            schema: Arc::new(Schema::new(vec![
                Field::new("hello", DataType::Utf8, false),
            ])),
            highest: 0,
        });

        let key_prefix = "t101_chello_r";
        let target = Arc::new(Schema::new(vec![
            Field::new("hell", DataType::Utf8, false),
        ]));

        let db = DB::open_default("./tmp").unwrap();
        let src = KVTable::new(&meta, Arc::new(db));
        for i in 0..100i32 {
            let key = format!("{key_prefix}{i:0>6}").into_bytes();
            src.db.put(key, format!("{i}").into_bytes()).unwrap();
        }

        let table_exec = DBTableScanExec::new(101, &target, &src);
        let result = table_exec.execute(0, Arc::new(TaskContext::default()));
        let output = match result {
            Ok(out) => out,
            Err(_) => {
                assert_eq!(9, 1);
                return;
            }
        };

        let mut stream = output;
        // Only 1 batch
        if let Some(batch) = stream.next().await {
            match batch {
                Ok(batch) => {
                    assert_eq!(batch.num_rows(), 100);

                    let col = batch.columns();
                    let col = col.get(0).unwrap();
                    for (i, row) in as_string_array(col).iter().enumerate() {
                        assert_eq!(row, Some(format!("{i}").as_str()))
                        // println!("{i} {}", row.unwrap());
                    }
                }
                _ => {}
            }
        }
    }
```

最后，对`KVTableSink`进行测试，目的是为了确保数据确实有通过`insert_into`插入表中：

```rust
    /// Create a `KVTable` with a single column and `insert into` it
    /// by `values`, check if the data is inserted
    #[tokio::test]
    async fn test_db_write() -> Result<()> {
        // Create a new schema with one field called "a" of type Int32
        let schema = Arc::new(Schema::new(vec![Field::new("a", DataType::Utf8, false)]));

        // Create a new batch of data to insert into the table
        let batch = RecordBatch::try_new(
            schema.clone(),
            vec![Arc::new(StringArray::from(vec!["hello", "world", "!"]))],
        )?;
        // Run the experiment and obtain the resulting data in the table
        let resulting_data_in_table =
            experiment(schema, vec![vec![batch.clone()]], vec![vec![batch.clone()]])
                .await?;
        // Ensure that the table now contains two batches of data in the same partition
        for col in resulting_data_in_table.columns() {
            let arr = as_string_array(col);
            assert_eq!(
                arr,
                &StringArray::from(vec!["hello", "world", "!", "hello", "world", "!"]),
            )
        }

        // todo: remove test table after this test
        Ok(())
    }

	/// This function create a table with `initial_data` to 
	/// insert `inserted_data` into and return the final batch of the table.
    async fn experiment(
        schema: SchemaRef,
        initial_data: Vec<Vec<RecordBatch>>,
        inserted_data: Vec<Vec<RecordBatch>>,
    ) -> Result<RecordBatch> {
        let expected_count: u64 = inserted_data
            .iter()
            .flat_map(|batches| batches.iter().map(|batch| batch.num_rows() as u64))
            .sum();

        // Create a new session context
        let session_ctx = SessionContext::new();
        // Create meta of a table
        let dest_meta = Arc::new(KVTableMeta {
            id: 1002,
            name: "Dest".to_string(),
            schema: Arc::new(Schema::new(Fields::from(vec![
                Field::new("a", DataType::Utf8, false),
            ]))),
            highest: 0,
        });
        // Create KV store
        let db = DB::open_default("tmp").unwrap();
        let db = Arc::new(db);
        // Create and register the initial table with the provided schema and data
        let initial_table = Arc::new(KVTable::try_new(&dest_meta, Arc::clone(&db), initial_data).await?);
        session_ctx.register_table("Dest", initial_table.clone())?;

        // Create source table meta
        let src_meta = KVTableMeta {
            id: 1001,
            name: "Src".to_string(),
            schema: Arc::new(Schema::new(Fields::from(vec![
                Field::new("a", DataType::Utf8, false),
            ]))),
            highest: 0,
        };

        let exprs = vec![
            vec![Expr::Literal(ScalarValue::Utf8(Some("hello".to_string())))],
            vec![Expr::Literal(ScalarValue::Utf8(Some("world".to_string())))],
            vec![Expr::Literal(ScalarValue::Utf8(Some("!".to_string())))],
        ];
        let values_plan = LogicalPlanBuilder::values(exprs)?
            .project(vec![Expr::Column("column1".into()).alias("a")])?
            .build()?;

        // Create an insert plan to insert the source data into the initial table
        let insert_into_table =
            LogicalPlanBuilder::insert_into(values_plan, "Dest", &schema, false)?.build()?;
        // Create a physical plan from the insert plan
        let plan = session_ctx
            .state()
            .create_physical_plan(&insert_into_table)
            .await?;

        // Execute the physical plan and collect the results
        let res = collect(plan, session_ctx.task_ctx()).await?;
        assert_eq!(extract_count(res), expected_count);

        let target_schema = Arc::new(Schema::new(Fields::from(vec![
            Field::new("a", DataType::Utf8, false),
        ])));
        let exec = DBTableScanExec::new(1002, &target_schema, &initial_table);
        let mut stream = exec.execute(0, session_ctx.task_ctx())?;
        if let Some(batch) = stream.next().await.transpose()? {
            Ok(batch)
        } else {
            Err(exec_datafusion_err!("unknown err when fetching batch from stream"))
        }
    }

	/// This function takes out the number from batches, and
	/// `res` should have only one row and one column.
    fn extract_count(res: Vec<RecordBatch>) -> u64 {
        assert_eq!(res.len(), 1, "expected one batch, got {}", res.len());
        let batch = &res[0];
        assert_eq!(
            batch.num_columns(),
            1,
            "expected 1 column, got {}",
            batch.num_columns()
        );
        let col = batch.column(0).as_primitive::<UInt64Type>();
        assert_eq!(col.len(), 1, "expected 1 row, got {}", col.len());
        let val = col
            .iter()
            .next()
            .expect("had value")
            .expect("expected non null");
        val
    }
```
完整代码可以参考[这里](https://github.com/Kev1n8/drdb/commit/62a552803d070be7db2381351013d021b99f6e1e)。