# Model / Message Structs

This document provides an overview of the various data structures (structs) used in our application, including models and messages. Each struct is defined with its fields and their respective types.

## Enums

### SourceType
Defines the types of data sources available.
```protobuf
enum SourceType {
    NAVER_SHOPPING = 0;
    COUPANG = 1;
    LOTTE_ON = 2;
}
```

### TaskType
Defines the types of tasks that can be performed.
```protobuf
enum TaskType {
    PRODUCT_DETAIL_CRAWL = 0;   // Task to crawl product details
    KEYWORD_TRACKING = 1;       // Task to track keyword trends
}
```

## Models

### Task
Represents a data collection task.
```protobuf
message Task {
    message Payload {
        string keyword = 1;          // Keyword to be tracked or searched
        string url = 2;              // URL for crawl task
        int32 max_pages = 3;         // Maximum number of pages to crawl
    }
  
    string task_id = 1;               // Unique identifier for the task (UUID)
    TaskType task_type = 2;           // Type of the task
    SourceType source_type = 3;       // Source from which data is collected
    Payload payload = 4;              // Payload containing task-specific parameters
    int64 created_at = 5;             // Timestamp of task creation
    int64 updated_at = 6;             // Timestamp of last task update
    int64 max_retries = 7 [default = 3]; // Maximum number of retries for the task
}
```
