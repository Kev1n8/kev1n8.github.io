+++
title = "macro_rules in Rust"
date = 2024-12-09
draft = true

[taxonomies]
tags = ["rust"]
+++


# Rust声明宏简单实践

记一次Rust声明宏简单实践。

## 需求

项目中的 `resources/src/api/apply.rs` 定义了资源的 `apply` 接口支持，在重构前，代码结构大致如下面所示
```rust
pub enum ApplySupportedResource {
    DataId(DataId),
    ResultTable(ResultTable),
    Databus(Databus),
    // 其他变体...
}

async fn apply(config: &ApplySupportedResource) -> Result<(), Error> {
    for config in data.config.iter() {
        let (namespaced_ref, validate_result): (NamespaceName, _) = match config {
            ApplySupportedResource::DataId(ref dataid) => (dataid.into(), dataid.validate(&())),
            ApplySupportedResource::ResultTable(ref rt) => (rt.into(), rt.validate(&())),
            // 其他cases...
        };

        if validate_result.is_err() {
            errors.push((namespaced_ref, validate_result.err().unwrap()));
        }
    }

    for config in data.config.iter() {
        match config {
            ApplySupportedResource::DataId(r) => {
                // Special op.
            }
            ApplySupportedResource::ResultTable(r) => {
                r.save(&context.db).await.context(DbErrSnafu {})?
            }
            // 其他cases同rt...
        }
    }

    api_ok!(())
}
```

可以看到，在 `apply` 函数中，每次需要对 `config` 进行操作时，都需要一个庞大的 `match` 来覆盖所有变体，并且大部分变体的操作都是一样的（调用相同函数）。因此，考虑使用宏来生成这些代码，简化代码工作量。

## 任务

观察两个 `match` 操作，分别做了 validate 和 save 操作，那么我们可以将这些操作隐藏到两个函数中：
- `fn validate_config(config: &ApplySupportedResource) -> (NamespaceName, Result<(), Report>)`
- `async fn save_config(context: &Arc<DabContext>, config: &ApplySupportedResource) -> Result<(), Error>`

希望达到的效果：只需在一处添加需要支持的资源类型，即可自动生成所有需要的代码。

Rust中有两种宏：
- 声明宏
- 过程宏

过程宏又分为：
- *function-like* -- `$name ! $arg` 类似声明宏
- *atribute* -- `#[$arg]`
- *derive* -- `#[derive(...)]`

在这里声明宏已经可以满足我们的需要，所以不考虑过程宏。

----
关于宏的选择的建议：
> Before deciding which type of macro to write, ask yourself the questions below:
> - Is this a basic find/replace (e.g. implementing the same trait for different integer types or tuple lengths) or do I need some sort of conditional logic?
> - Do I need to track more "state" than a can be achieved with a simple pushdown accumulator?
> - Is this purely a syntactic operation or do I need to inspect a token's value and do logic with it? 

来源 https://users.rust-lang.org/t/how-do-you-decide-when-to-use-procedural-macros-over-declarative-ones/58667/3

## 行动

首先了解声明宏的使用方法：
```rust
macro_rules! $name {
    $rule0 ;
    $rule1 ;
    // …
    $ruleN ;
}

$rule: ($matcher) => {$expansion}
```

先尝试写一个：
```rust
macro_rules! generate_support_resource {
    () => {
        #[derive(serde::Deserialize, Debug, serde::Serialize)]
        #[serde(tag = "kind")]
        pub enum ApplySupportedResource {
            // variants...
        }

        fn validate_config(config: &ApplySupportedResource) -> (NamespaceName, Result<(), Report>) {
            match config {
                // cases...
            }
        }

        async fn save_config(context: &Arc<DabContext>, config: &ApplySupportedResource) -> Result<(), Error> {
            // cases...
        }
    };
}
```

声明宏提供了重复匹配的方法： 
```rust
$ ( ... ) sep rep
```
`sep` 分隔符可选: `,`, `;` 或 space（默认）

`rep` 重复选项: `*`, `+`, 或 `?`; 其中`?`不和 `sep` 一起使用，因为至多 1 个

举个例子：
```rust
macro_rules! print_all {
    ($($e:ident),*) => {
        $(
            println!("{}", $e);
        )*
    }
}

fn main() {
    let (a, b, c, d, e) = (1, 2, 3, 4, 5);
    print_all!(a, b, c, d, e);
    // 这里则会展开为
    // println!("{}", a);
    // println!("{}", b); 
    // println!("{}", c); 
    // println!("{}", d); 
    // println!("{}", e); 
}
```

利用重复匹配，我们就可以使用宏对各个变体生成代码了：
```rust
macro_rules! generate_support_resource {
    // 使用标识符来接收变体列表
    ($($variant:ident),*) => {
        #[derive(serde::Deserialize, Debug, serde::Serialize)]
        #[serde(tag = "kind")]
        pub enum ApplySupportedResource {
            $(
                $variant($variant),
            )*
        }
        
        fn validate_config(config: &ApplySupportedResource) -> (NamespaceName, Result<(), Report>) {
            match config {
                $(
                    ApplySupportedResource::$variant(resource) => (Into::<NamespaceName>::into(resource), resource.validate(&())),
                )*
            }
        }

        async fn save_config(context: &Arc<DabContext>, config: &ApplySupportedResource) -> Result<(), Error> {
            // Wait, 似乎DataId和其他人不一样？
        }
    };
}
```

在第二个 `match` 也就是 save 操作的时候，`DataId` 是一个例外，那么我们就把它单拎出来，可能的一种方法是在参数指明例外情况：
```rust
macro_rules! generate_support_resource {
    // 用 `;` 来分割例外和其他变体列表
    ($exception:ident; $($variant:ident),*) => {
        #[derive(serde::Deserialize, Debug, serde::Serialize)]
        #[serde(tag = "kind")]
        pub enum ApplySupportedResource {
            $exception($exception),
            $(
                $variant($variant),
            )*
        }

        fn validate_config(config: &ApplySupportedResource) -> (NamespaceName, Result<(), Report>) {
            match config {
                // 因为单拎出来了，要记得匹配
                ApplySupportedResource::$exception(resource) => (Into::<NamespaceName>::into(resource), resource.validate(&())),
                $(
                    ApplySupportedResource::$variant(resource) => (Into::<NamespaceName>::into(resource), resource.validate(&())),
                )*
            }
        }

        async fn save_config(context: &Arc<DabContext>, config: &ApplySupportedResource) -> Result<(), Error> {
            match config {
                // 例外单独处理
                ApplySupportedResource::$exception(data_id) => {
                    // Special op.
                }
                // 列表的其他变体保持一致
                $(
                    ApplySupportedResource::$variant(resource) => {
                        resource.save(&context.db).await.context(DbErrSnafu {})?
                    }
                )*
            }
            Ok(())
        }
    }
}
```

## 效果

如果需要对一个新的资源支持 `apply`，只需要在 `generate_support_resource` 宏参数重添加元素即可。

----
参考PR：https://github.com/TencentBlueKing/bk-base/pull/3015/

