---
title: 'Паттерн Repository в Rust'
pubDate: '2026-06-10'
--- 

Бизнес логика не должна знать, где храняться данные. Он решает именно это: прячет SQL за трейтом, а сервис работает с абстракцией.

## Пример
```rust
use thiserror::Error;

#[derive(Debug, Clone)]
pub struct User {
    pub id: i64,
    pub email: String,
    pub username: String,
}

#[derive(Debug, Error)]
pub enum UserRepositoryError {
    #[error("not found")]
    NotFound,
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
}

#[async_trait::async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: i64) -> Result<User, UserRepositoryError>;
    async fn save(&self, user: &User) -> Result<User, UserRepositoryError>;
    async fn delete(&self, id: i64) -> Result<(), UserRepositoryError>;
}
```

## Реализация
```rust
use sqlx::PgPool;
use crate::domain::user::{User, UserRepository, UserRepositoryError};

pub struct PostgresUserRepository {
    pool: PgPool,
}

impl PostgresUserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait::async_trait]
impl UserRepository for PostgresUserRepository {
    async fn find_by_id(&self, id: i64) -> Result<User, UserRepositoryError> {
        sqlx::query_as!(
            User,
            "SELECT id, email, username FROM users WHERE id = $1",
            id
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| match e {
            sqlx::Error::RowNotFound => UserRepositoryError::NotFound,
            other => UserRepositoryError::Database(other),
        })
    }

    async fn save(&self, user: &User) -> Result<User, UserRepositoryError> {
        sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (id, email, username)
            VALUES ($1, $2, $3)
            ON CONFLICT (id) DO UPDATE
                SET email = EXCLUDED.email,
                    username = EXCLUDED.username
            RETURNING id, email, username
            "#,
            user.id,
            user.email,
            user.username,
        )
        .fetch_one(&self.pool)
        .await
        .map_err(UserRepositoryError::Database)
    }

    async fn delete(&self, id: i64) -> Result<(), UserRepositoryError> {
        let result = sqlx::query!("DELETE FROM users WHERE id = $1", id)
            .execute(&self.pool)
            .await
            .map_err(UserRepositoryError::Database)?;

        if result.rows_affected() == 0 {
            return Err(UserRepositoryError::NotFound);
        }

        Ok(())
    }
}
```

## Сервис
```rust
use std::sync::Arc;
use crate::domain::user::{User, UserRepository, UserRepositoryError};

pub struct UserService {
    repository: Arc<dyn UserRepository>,
}

impl UserService {
    pub fn new(repository: Arc<dyn UserRepository>) -> Self {
        Self { repository }
    }

    pub async fn change_email(
        &self,
        id: i64,
        new_email: String,
    ) -> Result<User, UserRepositoryError> {
        let mut user = self.repository.find_by_id(id).await?;
        user.email = new_email;
        self.repository.save(&user).await
    }
}
```

## Итог

Ни строчки SQL в бизнес логике. Нужна другая база данных - пишешь новую реализацию трейта, сервис не трогаешь.
