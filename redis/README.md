# mobc_redis

## Compatibility

Things got to be a little strange, the async-API of `redis-rs` is built with tokio 0.1, and they can run in a runtime built with async-std! But tokio-postgres which is built with tokio 0.2  is not lucky. When redis-rs has released a new version bases on tokio 0.2, it will also not compatible with async-std

## Cargo Features

```toml
[features]
default = ["tokio-runtime"]
tokio-runtime = ["mobc/tokio-runtime"]
async-std-runtime = ["mobc/async-std-runtime"]
```

## Examples

```rust
use mobc::futures::channel::mpsc;
use mobc::futures::compat::Future01CompatExt;
use mobc::futures::prelude::*;
use mobc::runtime::DefaultExecutor;
use mobc::Error;
use mobc::Pool;
use mobc_redis::redis::{self, RedisError};
use mobc_redis::RedisConnectionManager;
use std::time::Instant;

const MAX: usize = 5000;

async fn single_request(
    pool: Pool<RedisConnectionManager<DefaultExecutor>>,
    mut sender: mpsc::Sender<()>,
) -> Result<(), Error<RedisError>> {
    let mut conn = pool.get().await?;
    let raw_redis_conn = conn.take_raw_conn();

    let (raw_redis_conn, pong) = redis::cmd("PING")
        .query_async::<_, String>(raw_redis_conn)
        .compat()
        .await?;

    conn.set_raw_conn(raw_redis_conn);

    assert_eq!("PONG", pong);
    sender.send(()).await.unwrap();
    Ok(())
}

async fn do_redis(sender: mpsc::Sender<()>) -> Result<(), Error<RedisError>> {
    let client = redis::Client::open("redis://127.0.0.1").unwrap();
    let manager = RedisConnectionManager::new(client);
    let pool = Pool::builder().max_size(40).build(manager).await?;

    for _ in 0..MAX {
        let pool = pool.clone();
        let tx = sender.clone();
        tokio::spawn(single_request(pool, tx).map(|_| ()));
    }
    Ok(())
}

#[tokio::main]
async fn main() {
    env_logger::init();
    let mark = Instant::now();
    let (tx, mut rx) = mpsc::channel::<()>(MAX);
    do_redis(tx).await.unwrap();

    let mut num: usize = 0;
    while let Some(_) = rx.next().await {
        num += 1;
        if num == MAX {
            break;
        }
    }

    println!("cost {:?}", mark.elapsed());
}

```