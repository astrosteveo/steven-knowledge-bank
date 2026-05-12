# MarkLogic integration with Semaphore

## Overview

- Semaphore Concepts Service
	- Feature in **Semaphore 5.8+** that uses MarkLogic Server as its backend to store and serve semantic models, improving query performance, and enables downstream systems to access models via JSON objects or GraphQL.
	- Provides a highly-available read-only version of the published models
	- Provides similar functionality as Semantic Enhancement Server (SES), but is not a drop-in replacement due to the two API's not being the same.
	- SCS API is available via **REST API calls** and **GraphQL**.
- MarkLogic Flux
	- Tool that can load and classify data by communicating with Semaphore while importing that automatically assigns tags/categories to documents as they are ingested into MarkLogic.
-  Semantic Metadata Hub
	- Semaphore enriches 

## MarkLogic Flux

## Description

- Can be used to automate common data movement use cases, such as:
	- Importing rows from RBDMS
	- Used to import JSON, XML, CSV, Parquet and other file types from a local file system, or S3 object storage
	- Extract text from binary documents, and classify them using Progress Semaphore
- Built on top of Apache Spark
- 

## Requirements

- Java 17 or Java 21
- MarkLogic 10.0-9+
- A MarkLogic REST API app server to connect Flux to

