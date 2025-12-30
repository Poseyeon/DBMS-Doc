---
layout: single
title: Beyond SQL
permalink: /beyond-sql/

toc: true
toc_sticky: true
sidebar:
  nav: "docs"
---

## Introduction

IsentaDB provides several custom keywords that are not part of the ISO/IEC 9075 specification. These extensions are designed to simplify common data inspection and management tasks.

## SHOW TABLES

The `SHOW TABLES` keyword is used to list all tables currently available in the database.

## INSPECT

The `INSPECT <table_name>` keyword displays all columns of a table along with their corresponding data types.

**Example:**

```yml
isenta> SHOW TABLES
Tables:
- USERS
- ORDERS
- PRODUCTS
```

**Example:**

```yml
isenta> INSPECT USERS
Table: USERS
----------------
Column               | Type
---------------------+----------------
ID                   | INT
NAME                 | TEXT
```

## GET

The `GET <table_name> AS <format>` keyword is used to list all the data from a table in a specified data format. Supported Format is JSON.

**Example:**

```yml
isenta> GET USERS AS JSON
{
  "name": "USERS",
  "columns": [
    {
      "name": "ID",
      "data_type": "INT"
    },
    {
      "name": "NAME",
      "data_type": "TEXT"
    }
  ],
  "rows": [
    {
      "values": [
        "1",
        "John Doe"
      ]
    },
    {
      "values": [
        "2",
        "Jane Doe"
      ]
    }
  ]
}
```