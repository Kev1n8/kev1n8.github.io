+++
title = "Extending DataFusion: Custom Query Plans & Optimizer Rules"
date = 2024-09-21

[taxonomies]
tags = ["rust", "datafusion"]
+++



## 扩展Datafusion之自定义计划

上一篇扩展datafusion介绍了通过自定义的 `TableProvider`, `ExecutionPlan` 和 `DataSink` 来帮助我们达到下面的目的：

1. 使用上下文注册 `TableProvider` 定向 `scan` 和 `Insert_into` 执行时的调用的对象（`ExecutionPlan`）
2. 自定义的 `ExecutionPlan` 执行从rocksdb读取表数据的操作
3. 自定义的 `DataSink`，提供 `write_all` 函数供datafusion的 `DataSinkExec` 调用，后者在 `insert_into` 的时候被创建

而本文也将从几个要自定义实现的 `trait` 开始介绍，分别是：

- `QueryPlanner` -- 专门用于调用各 `Plannner` 生成 `LogicalPlan` 和 `ExecutionPlan`，本文用于注册自定义的 `ExtensionPlanner`
- `OptimizerRule` -- 用于自定义一个优化规则，本文实现一个规则将datafusion自带的 `DML::DELETE` `PlanNode` 替换为自定义的 `DeletePlanNode`
- `UserDefinedLogicalNodeCore` -- 用于实现自定 `LogicPlanNode` 
- `ExtensionPlanner` -- 用于根据输入 `LogicPlanNode` 创建一个 `ExecutionPlan`

- `ExecuionPlan` -- 类似上一篇，只不过本文将实现 `Delete` 逻辑

## QueryPlanner

这是 `SessionState` 的一个成员，我们可以看看如果不为其添加任何扩展，它会如何初始化：

### 初始化路径源码一览

首先是出处，`query_planner` 作为成员，需要满足 `QueryPlanner` 动态分发：

```rust
pub struct SessionState {
    ...
	/// Responsible for planning `LogicalPlan`s, and `ExecutionPlan`
    query_planner: Arc<dyn QueryPlanner + Send + Sync>,
    ...
}
```

其次，我们看 `SessionStateBuilder`，这里省略了其它属性设置 ：

```rust
impl SessionStateBuilder {
	pub fn build(self) -> SessionState {
        let Self {
            ...
            query_planner,
            ...
        } = self;

        let mut state = SessionState {
            ...
            // 如果没有设定好的，则使用DefaultQueryPlanner
            query_planner: query_planner.unwrap_or(Arc::new(DefaultQueryPlanner {})),
			...
        };
    	...
        state
    }
}
```

没有规定一定要通过Builder来设置 `SessionState`，重要的是我们知道了Datafusion提供了默认的 `QueryPlanner`。

接着，来看这个 `DefaultQueryPlanner`：

```rust
struct DefaultQueryPlanner {}

#[async_trait]
impl QueryPlanner for DefaultQueryPlanner {
    /// Given a `LogicalPlan`, create an [`ExecutionPlan`] suitable for execution
    async fn create_physical_plan(
        &self,
        logical_plan: &LogicalPlan,
        session_state: &SessionState,
    ) -> datafusion_common::Result<Arc<dyn ExecutionPlan>> {
        let planner = DefaultPhysicalPlanner::default();
        planner
            .create_physical_plan(logical_plan, session_state)
            .await
    }
}
```

可见，datafusion又把初始化交给了 `DefaultPhysicalPlanner`：

```rust
#[derive(Default)]
pub struct DefaultPhysicalPlanner {
    // 如果我们注册了扩展，则会保存在这里
    extension_planners: Vec<Arc<dyn ExtensionPlanner + Send + Sync>>,
}

#[async_trait]
impl PhysicalPlanner for DefaultPhysicalPlanner {
    /// Create a physical plan from a logical plan
    async fn create_physical_plan(
        &self,
        logical_plan: &LogicalPlan,
        session_state: &SessionState,
    ) -> Result<Arc<dyn ExecutionPlan>> {
        // 具体的创建plan流程
        match self.handle_explain(logical_plan, session_state).await? {
            Some(plan) => Ok(plan),
            None => {
                let plan = self
                    .create_initial_plan(logical_plan, session_state)
                    .await?;

                self.optimize_physical_plan(plan, session_state, |_, _| {})
            }
        }
    }
    ...
}
```

到这里我们就知道，`QueryPlanner` 起到一个根据 `LogicalPlan` 创建 `PhysicalPlan` (`ExecutionPlan`) 的作用，具体的计划创建过程（关于从 `LogicalPlanNode` 到 `ExecutionPlan` 可以参考后续 `Planner` 的定义）就不展开了。

### 如何自定义

至于自定义的 `QueryPlanner`， 就是简单添加一个自定义的 `Planner` 到 `DefaultPhysicalPlanner` 中去：

```rust
struct DeleteQueryPlanner {
    table_id: u64,
    db: Arc<DB>,
}

#[async_trait]
impl QueryPlanner for DeleteQueryPlanner {
    async fn create_physical_plan(
        &self,
        logical_plan: &LogicalPlan,
        session_state: &SessionState,
    ) -> Result<Arc<dyn ExecutionPlan>> {
        // 这里并非使用默认，而是添加了一个自定义的DeletePlanner
        let planner = DefaultPhysicalPlanner::with_extension_planners(vec![Arc::new(
            DeletePlanner {
                table_id: self.table_id,
                db: Arc::clone(&self.db),
            },
        )]);

        planner
            .create_physical_plan(logical_plan, session_state)
            .await
    }
}
```

这样一来，后续的sql操作在转换计划的时候，注册了这个 `QueryPlanner` 的 `SessionState` 就知道 `DeletePlanner`，那么在遇到 `DeleteLogicalPlanNode` 的时候也就知道如何生成相应的 `ExecutionPlan`，也就支持了 `Delete` 操作。

那么问题来了，怎么把Datafusion中原有的 `DML::Delete` 这个 `PlanNode` 替换成真正的，自定义的 `PlanNode` 呢？

## OptimizerRule

顾名思义，这个 `trait` 可以让我们自定义一个逻辑优化器规则。本文要做的，就是将datafusion解析 `DELETE` 操作产生的 `LogicPlan::DML(DELETE)`  给换成我们自己的实现，前者在执行阶段会返回错误：

```rust
impl DefaultPhysicalPlanner {
    ...
    async fn map_logical_node_to_physical(
        &self,
        node: &LogicalPlan,
        session_state: &SessionState,
        children: ChildrenContainer,
    ) -> Result<Arc<dyn ExecutionPlan>> {
        ...
        let exec_node = match node {
        	LogicalPlan::Extension(Extension { node }) => {
                let mut maybe_plan = None;
                let children = children.vec();
                for planner in &self.extension_planners {
                    if maybe_plan.is_some() {
                        break;
                    }

                    let logical_input = node.inputs();
                    maybe_plan = planner
                        .plan_extension(
                            self,
                            node.as_ref(),
                            &logical_input,
                            &children,
                            session_state,
                        )
                        .await?;
                }

                let plan = match maybe_plan {
                        Some(v) => Ok(v),
                        _ => plan_err!("No installed planner was able to convert the custom node to an execution plan: {:?}", node)
                    }?;

                // Ensure the ExecutionPlan's schema matches the
                // declared logical schema to catch and warn about
                // logic errors when creating user defined plans.
                if !node.schema().matches_arrow_schema(&plan.schema()) {
                    return plan_err!(
                            "Extension planner for {:?} created an ExecutionPlan with mismatched schema. \
                            LogicalPlan schema: {:?}, ExecutionPlan schema: {:?}",
                            node, node.schema(), plan.schema()
                        );
                } else {
                    plan
                }
            }
        ...
			LogicalPlan::Dml(dml) => {
                // DataFusion is a read-only query engine, but also a library, so consumers may implement this
                return not_impl_err!("Unsupported logical plan: Dml({0})", dml.op);
            }
        ...
        }
    }
    ...
}
```

从上面的代码可以看见，如果我们没有用我们自己的实现（也就是 `LogicalPlan::Extension`）替换 `LogicalPlan::Dml(dml)`，就会返回错误。而如果通过我们自定义的rule将这个计划结点换成，就可以执行我们自己的实现（对应 `DeleteLogicalPlanNode`）。

实现 `OptimizerRule` 关键是实现 `re_write` 方法，这个方法检查当前结点是否为目标结点（`LogicalPlan::Dml(delete)`），如果是的话就用自定义结点取代（`DeleteLogicalPlanNode`），代码如下：

```rust
struct DeleteReplaceRule {}

impl OptimizerRule for DeleteReplaceRule {
    fn rewrite(
        &self,
        plan: LogicalPlan,
        _config: &dyn OptimizerConfig,
    ) -> Result<Transformed<LogicalPlan>, DataFusionError> {
        if let LogicalPlan::Dml(DmlStatement {
            table_name,
            op: WriteOp::Delete,
            input,         // table source
            output_schema, // single count
            ..
        }) = plan
        {
            // input should have only 1 item, and
            // it can only be `LogicalPlan::Filter`, or
            // `LogicalPlan::Scan`?
            let _name = table_name.to_string();
            let _schema = input.schema().to_string();
            let _inputt = input.as_ref().to_string();
            Ok(Transformed::yes(LogicalPlan::Extension(Extension {
                node: Arc::new(DeletePlanNode {
                    input: input.as_ref().clone(),
                    schema: output_schema,
                    expr: input.expressions(),
                }),
            })))
        } else {
            Ok(Transformed::no(plan))
        }
    }
    ...
}
```

## UserDefinedLogicalNodeCore

直接贴代码：

```rust
struct DeletePlanNode {
    input: LogicalPlan,
    schema: DFSchemaRef,
    expr: Vec<Expr>,
}

impl UserDefinedLogicalNodeCore for DeletePlanNode {
    fn with_exprs_and_inputs(
        &self,
        exprs: Vec<Expr>,
        mut inputs: Vec<LogicalPlan>,
    ) -> Result<Self> {
        assert_eq!(inputs.len(), 1, "input size inconsistent");
        Ok(Self {
            input: inputs.swap_remove(0),  // 假定只有一个子节点
            schema: Arc::clone(&self.schema),
            expr: exprs,
        })
    }
}
```

实际运行时，这个函数会被调用很多次，但都没有任何改变。To be found...

## ExtensionPlanner

重头戏之一，我们在上面已经实现了 `QueryPlanner`, `OptimizerRule`, `PlanNode`，这三者结合已经能够做到下面的事情：

1. 为 `SessionState` 注册 `QueryPlanner` 和 `OptimizerRule`
2. sql执行DML之 `DELETE` 操作的时候，生成的逻辑计划将是自定义的逻辑计划

但是，`DefaultPhysicalPlanner` 并不知道我们自定义的逻辑计划要生成什么样的执行计划。所以，`ExtensionPlanner` 就要站出来发挥作用了。

```rust
struct DeletePlanner {
    table_id: u64,
    db: Arc<DB>,
}

#[async_trait]
impl ExtensionPlanner for DeletePlanner {
    async fn plan_extension(
        &self,
        _planner: &dyn PhysicalPlanner,
        node: &dyn UserDefinedLogicalNode,
        logical_inputs: &[&LogicalPlan],
        physical_inputs: &[Arc<dyn ExecutionPlan>],
        _session_state: &SessionState,
    ) -> Result<Option<Arc<dyn ExecutionPlan>>> {
        let node =
            if let Some(delete_node) = node.as_any().downcast_ref::<DeletePlanNode>() {
                assert_eq!(logical_inputs.len(), 1);
                assert_eq!(physical_inputs.len(), 1);
                let output_schema = Arc::new(Schema::from(&(*delete_node.schema)));
                let input_schema = Arc::clone(&physical_inputs[0].schema());
                Some(Arc::new(DeleteExec::new(
                    self.table_id,
                    &self.db,
                    &input_schema,
                    &output_schema,
                    &physical_inputs[0],
                )))
            } else {
                None
            };
        Ok(node.map(|exec| exec as Arc<dyn ExecutionPlan>))
    }
}
```

逻辑就是：查看当前逻辑结点是否为“我”负责的，如果是的话则生成对应执行计划，否则返回None，datafusion会自动寻找其它对应的 `Planner`。

## ExecutionPlan

终于回到我们的 `KVTable` 了，我们在这里实现真正的删除操作，目的是实现从表中（db中）删除对应的行，而这些行会通过 `input` 过滤进来。每个行有一个唯一标识，即 `row_id`，删除操作会遍历表的每个列，然后遍历每个要删除的 `row_id`，这样可以构造出目标 `key`， `t[tid]_c[name]_r[rid]`。

实现如下：

```rust
    /// This method takes in the batch that is supposed to be deleted, and
    /// delete them from the table of self.table_id, and
    /// return rows deleted
	fn delete_batch_from_table(&self, batch: RecordBatch) -> Result<u64> {
        let target = batch.columns().last().unwrap(); // by design, the `row_id` is the last column

        // Iter over the `target` array to get each `row_id` to delete
        let mut cnt = 0u64;
        let target = as_string_array(target)?;
        for field in self.in_schema.fields.iter() {
            let name = field.name();
            if name.eq("row_id") {
                continue; // break
            }
            for id in target {
                match id {
                    None => {
                        todo!()
                    }
                    Some(id) => {
                        cnt += 1;
                        // TODO: Change type of `row_id` from Utf8 to u64
                        let key = make_row_key(self.table_id, name, id.parse().unwrap());
                        self.db.delete(key).unwrap();
                    }
                }
            }
        }
        Ok(cnt)
    }
```

至此，我们的 `KVTable` 就支持删除操作了。

## BONUS：测试

同样的，贴一份基础测试，验证删除的有效性：

```rust
	#[tokio::test]
    async fn basic_delete_test() {
        let db = Arc::new(DB::open_default("./tmp").unwrap());

        // Registering the custom planners and rule
        let config = SessionConfig::new().with_target_partitions(1);
        let runtime = Arc::new(RuntimeEnv::default());
        let state = SessionStateBuilder::new()
            .with_config(config)
            .with_runtime_env(runtime)
            .with_default_features()
            .with_query_planner(Arc::new(DeleteQueryPlanner {
                table_id: 1003,
                db: Arc::clone(&db),
            }))
            .with_optimizer_rule(Arc::new(DeleteReplaceRule {}))
            .build();
        let ctx = SessionContext::new_with_state(state);
```

首先，我们创建一个 `SessionState` ，并注册自定义的 `Rule` 和 `Planner`，并根据这个State创建Datafusion的上下文。

```rust
        // Create table for testing and register
        let schema = Arc::new(Schema::new(Fields::from(vec![
            Arc::new(Field::new("c1", DataType::Utf8, false)),
            Arc::new(Field::new("row_id", DataType::Utf8, false)),
        ])));
        let table_meta = Arc::new(KVTableMeta {
            id: 1003,
            name: "t".to_string(),
            schema: Arc::clone(&schema),
            highest: 0, // default, unknown
        });
        let table_provider = Arc::new(KVTable::new(&table_meta, Arc::clone(&db)));

        // Since create table is not used here, we have to init the meta of table in db first
        let meta_key = table_meta.make_key();
        let meta_val = table_meta.make_value();
        db.put(meta_key, meta_val).unwrap();
        ctx.register_table("t", table_provider).unwrap();
```

接下来，创建表，并注册 `TableProvider` 到上下文。因为还没有实现 `CREATE` DDL，所以这里手动往 `db` 中添加表的元数据。

```rust
		// Now insert table with init data
        let inserted = ctx
            .sql("INSERT INTO t VALUES('hello', '001'), ('world', '002'), ('!', '003')")
            .await
            .unwrap();
        // Should have inserted 3 row
        let bind = inserted.collect().await.unwrap();
        let batch = bind.first().unwrap();
        let t = as_uint64_array(&batch.columns()[0]).unwrap();
        assert_eq!(t.value(0), 3);

        // Now try to delete from the table
        let deleted = ctx.sql("DELETE FROM t where c1='!'").await.unwrap();
        let _plan = deleted.logical_plan().to_string();
        let bind = deleted.collect().await.unwrap();
        let batch = bind.first().unwrap();
        let t = as_uint64_array(&batch.columns()[0]).unwrap();
        // Should have deleted 1 row
        assert_eq!(t.value(0), 1);
```

创建完表后，利用之前实现的 `insert_into` 插入一些数据（这里有一个隐藏的问题，因为 `row_id` 是显式地在表schema中的，插入时其实并不知道具体的 `row_id`，目前的方法是随意输入值填充，但是后期的话应该要考虑让这个列对用户来说是透明的），插入后尝试删除行。

```rust
		// Now scan the table and compare
        let expected = RecordBatch::try_new(
            Arc::clone(&schema),
            vec![
                Arc::new(StringArray::from(vec!["hello", "world"])),
                Arc::new(StringArray::from(vec!["000001", "000002"])),
            ],
        )
        .unwrap();
        let df = ctx.sql("SELECT c1 FROM t").await.unwrap();
        let bind = df.collect().await.unwrap();
        let res = pretty_format_batches(bind.as_slice()).unwrap();
        println!("{}", res);
        let out = bind.first().unwrap();
        assert_eq!(out, &expected);
    }
```

删除成功后，执行 `scan` 操作，查看输出是否和预期相同。