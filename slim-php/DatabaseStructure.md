# MySQL Database Structure

This document outlines the MySQL database schema design, UUID generation, and password hashing policy for a Slim Framework application, based on the Slim Framework documentation and project requirements.

## Instructions

- Design a `users` table with:
  - `id`: CHAR(36), primary key, for UUIDs.
  - `username`: VARCHAR(50), unique, not null.
  - `password`: VARCHAR(255), not null, for hashed passwords.
  - `email`: VARCHAR(100), unique, not null.
  - `created_at`: TIMESTAMP, default to current timestamp.
- Use InnoDB engine and `utf8mb4` charset.
- Create a SQL script in `database.sql` for the table structure.
- Implement password hashing:
  - Use PHP’s `password_hash()` with `PASSWORD_BCRYPT`.
  - Enforce minimum password requirements (e.g., 8 characters, including letters and numbers) in the user creation logic.
- Generate UUIDs using `ramsey/uuid` package. Use `Uuid::uuid4()->toString()` for each new record.

## Example Code

### Database Schema
```sql
CREATE TABLE users (
    id CHAR(36) PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### User Model with UUID and Password Hashing
```php
<?php

namespace App\Models;

use Ramsey\Uuid\Uuid;

class User
{
    public string $id;
    public string $username;
    public string $password;
    public string $email;

    public function __construct(array $data)
    {
        $this->id = Uuid::uuid4()->toString();
        $this->username = $data['username'];
        $this->password = password_hash($data['password'], PASSWORD_BCRYPT);
        $this->email = $data['email'];
    }
}
```